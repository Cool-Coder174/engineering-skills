# The Eight Stability Patterns

**Sources:** *Release It! Design and Deploy Production-Ready Software*, Michael T. Nygard (Pragmatic
Bookshelf, 2007). Chapter 5, "Stability Patterns", Sec. 5.1 through Sec. 5.8, pp. 110–143. Chapter 6,
"Stability Summary", pp. 144–145. Page numbers are the PDF page numbers of the extracted source.

Chapter 4 names the ways a system breaks. Chapter 5 names the eight defenses. Read
`02-stability-antipatterns.md` to name the failure. Read this file to choose the defense. Each section
gives the signal, the mechanism, the rules with their pages, and the war story that the book tells.

**The count of patterns is not a quality measure.** Nygard states that "count of patterns applied" is
never a good quality metric (Ch. 5, p. 110). Aim for a recovery-oriented mind-set, not for coverage.

---

## 1. Which pattern applies in which direction

Nygard states the split. Timeouts apply to outbound requests. Fail Fast applies to incoming requests.
He calls the two patterns two sides of the same coin (Sec. 5.1, p. 114). Outbound means that we call a
vendor. Inbound means that a caller calls us. A vendor integration needs both halves.

| Pattern | Direction | What it does | Source |
|---|---|---|---|
| Use Timeouts | Outbound | Stops the wait for an answer that may never arrive | Sec. 5.1, p. 111 |
| Circuit Breaker | Outbound | Prevents the call while the dependency is sick | Sec. 5.2, p. 115 |
| Bulkheads | Both | Contains the damage inside one partition | Sec. 5.3, p. 119 |
| Steady State | Internal | Recycles every resource that the system accumulates | Sec. 5.4, p. 124 |
| Fail Fast | Inbound | Refuses work that the system cannot finish | Sec. 5.5, p. 131 |
| Handshaking | Inbound | Tells the caller to send less | Sec. 5.6, p. 134 |
| Test Harness | Test | Produces the out-of-spec faults that a mock cannot produce | Sec. 5.7, p. 136 |
| Decoupling Middleware | Architecture | Removes the shared clock between caller and callee | Sec. 5.8, p. 141 |

---

## 2. Use Timeouts

**Sec. 5.1, pp. 111–114. Outbound.**

**Signal.** The code contains a blocking wait with no time bound. A thread dump shows threads inside a
socket read. A vendor client library exposes no timeout setting.

**Mechanism.** The timeout stops the wait for an answer once you decide that the answer will not arrive.
On expiry the caller abandons the wait and returns an answer. A well-placed timeout gives fault
isolation, so a problem in another system does not have to become your problem. Nygard also writes that
"Hope is not a design method" (Sec. 5.1, p. 111).

- **Bound every wait that can hang.** A socket connect needs a connect timeout. A socket read needs a
  read timeout (Sec. 5.1, p. 111). A resource pool needs a timeout on checkout and on return, which
  unblocks the thread whether a resource arrives or not. Never use the no-argument form of `wait()`,
  `poll()`, `offer()`, or `tryLock()`. One database interaction hides three hangs. The checkout, the
  query, and the return of the connection can each hang (Sec. 5.1, p. 112).
- **A vendor client library often hides the socket.** Such a library prevents the application from
  setting vital timeouts (Sec. 5.1, p. 111). Audit every vendor SDK. See `01-integration-points.md`.
- **Collect the interaction into one place.** A QueryObject holds the part that changes. A Gateway holds
  connection handling, error handling, and result processing. The Gateway also makes the Circuit Breaker
  easy to apply (Sec. 5.1, pp. 112–113).
- **Do not retry at once.** Problems inside a data center last for a while, so a fast retry is very
  likely to fail again. Queue the work and retry it later (Sec. 5.1, p. 113).
- **On timeout, return an answer.** Return a failure, a success, or a note that the work is queued. For a
  website that uses service-oriented architectures, "fast enough" is probably under 250 milliseconds
  (Sec. 5.1, p. 113).
- **A timeout is a weak answer to an unbounded result set.** Nygard calls it a stop-gap and not much more
  (Sec. 5.1, p. 114). Bound the result set at the source.

**The war story.** Nygard ported the BSD sockets library to a mainframe UNIX environment with the RFCs
and the UNIX SVR4 source. The networking code was riddled with error handling for many flavors of
timeout, and he took the significance of timeouts from that work. The related sidebar concedes the cost.
Timeout handling can double the code, and that error-handling code adds resilience when a team writes it
well (Sec. 5.1, pp. 111–112). Catalog codes `I-01`, `I-02`, `I-05`, `I-08`. Retry rules live in
`05-overload-and-load-shedding.md`.

---

## 3. Circuit Breaker

**Sec. 5.2, pp. 115–118. Outbound.**

**Signal.** The code keeps calling a dependency that keeps failing. Threads accumulate inside one
integration point. Timeouts repeat at a steady rate.

**Mechanism.** Wrap the dangerous operation with a component that can circumvent the call when the
system is not healthy (Sec. 5.2, p. 115). The analogy is the household circuit breaker. That device lets
one circuit fail without damage to the house, and an operator can reset it afterwards.

**A circuit breaker prevents operations. It does not repeat them.** This is the exact difference between
a breaker and a retry (Sec. 5.2, p. 115). A retry sends the same request again. A breaker stops the
request from leaving at all. Never call a breaker a retry mechanism. Never let one component do both.

### The state machine

Three states exist. They are Closed, Open, and Half-Open. The table lists every transition in Figure 5.1
on p. 116.

| State | Event | The breaker does this | Next state |
|---|---|---|---|
| Closed | A call arrives | It executes the real operation | Closed |
| Closed | The call succeeds | It resets the failure count | Closed |
| Closed | The call fails | It counts the failure | Closed |
| Closed | The count reaches the threshold | It trips | Open |
| Open | A call arrives | It fails the call at once, with no attempt at the real operation | Open |
| Open | The reset timeout elapses | It attempts a reset | Half-Open |
| Half-Open | A call arrives | It executes the real operation once | Half-Open |
| Half-Open | The trial call succeeds | It resets | Closed |
| Half-Open | The trial call fails | It trips | Open |

Half-Open admits exactly one trial call (Sec. 5.2, p. 116). The threshold can count frequency instead of
a raw number. Nygard names the Leaky Bucket pattern as an implementation of that counter (Sec. 5.2,
p. 116, footnote 3).

- **Throw a different exception when the circuit is open.** The calling code then handles the open state
  differently from an ordinary failure (Sec. 5.2, p. 116). Catalog code `I-17`.
- **Track failure types separately.** Use a lower threshold for "timeout calling remote system" than for
  "connection refused" (Sec. 5.2, p. 117). Give each integration point its own breaker, because one
  shared breaker hides which dependency is sick. Catalog code `I-16`.
- **Decide the open-state behavior with the stakeholders.** A breaker degrades function automatically,
  and that choice affects the business (Sec. 5.2, p. 117). Do you accept an order with no stock
  confirmation? Write the answer down before the outage. See `09-vendor-slas-and-degradation.md`.
- **Log every state change. Expose the current state for query and monitoring** (Sec. 5.2, p. 117).
  Chapter 17, Transparency, p. 265, holds the detail. Chart the frequency of state changes, which Nygard
  calls a leading indicator of problems elsewhere. Use it in `04-fault-localization.md`.
- **Operations needs a way to trip or reset the breaker directly** (Sec. 5.2, p. 117). Catalog code
  `I-18`. That manual control is also the kill switch.
- **A popped breaker always means a serious problem.** Report it, record it, trend it, and correlate it
  (Sec. 5.2, p. 118). A breaker guards against Integration Points, Cascading Failures, Unbalanced
  Capacities, and Slow Responses.
- **Modern.** A sidecar proxy or a service mesh can hold the breaker outside the process. The rules above
  do not change.

---

## 4. Bulkheads

**Sec. 5.3, pp. 119–123. Both directions.**

**Signal.** Two consumers share one pool, one service, or one host. One consumer can consume the whole
resource. A single fault reaches every feature.

**Mechanism.** Partition the system so that one breach does not sink the ship. The bulkhead enforces a
principle of damage containment (Sec. 5.3, p. 119).

| Partition unit | What it contains | Source |
|---|---|---|
| Separate server farms per client | A load spike or a defect in one client | Sec. 5.3, pp. 119–121 |
| Virtual machines | A host-level fault, with pooled efficiency retained | Sec. 5.3, p. 121 |
| CPU binding | A process that consumes every cycle on the host | Sec. 5.3, p. 122 |
| Separate thread groups in one process | A blocked integration point in the application | Sec. 5.3, p. 122 |
| A reserved pool of administrative threads | The loss of all diagnostic access | Sec. 5.3, p. 122 |

- **Give each integration point its own resource pool.** One shared pool lets a slow vendor consume every
  thread. Catalog codes `I-19` and `I-20`.
- **Reserve threads for administration.** The server then still answers when every request-handling
  thread hangs, and it can collect data for the post-mortem (Sec. 5.3, p. 122).
- **Partitioning costs capacity.** A shared pool needs less total capacity than the sum of the separate
  pools when the peaks do not coincide (Sec. 5.3, p. 121).
- **Choose the boundary from business impact.** Examine the impact to the business of each loss of
  capability. Cross-reference those impacts against the architecture. Boundaries can align with the
  callers, with the functions, or with the topology (Sec. 5.3, p. 121). Test for maximum overall
  throughput, not for the best response time from an individual server (Sec. 5.3, p. 123).

**The war story.** Foo and Bar both depend on the enterprise service Baz (Figure 5.2, p. 120). If user
load crushes Foo, or a defect in Foo triggers a bug in Baz, then Bar and its users suffer too. The link
is unseen, so a performance problem in Bar is very hard to diagnose. A maintenance window for Baz needs
coordination with both clients. Figure 5.3 on p. 120 partitions Baz into one pool per client and breaks
both the failure link and the diagnosis link. Nygard has also seen one process that went berserk consume
an eight-CPU server (Sec. 5.3, p. 122). Bound to one CPU, that process could consume cycles only on that
CPU. CPU binding looks like a performance tuning step. It is really a bulkhead.

---

## 5. Steady State

**Sec. 5.4, pp. 124–130. Internal.**

**Signal.** An operator connects to a production server to clear disk space. A nightly restart exists.
Latency rises at constant load over weeks.

**Mechanism.** Nygard states the law. For every mechanism that accumulates a resource, some other
mechanism must recycle that resource. He calls the accumulation sludge. Keep people off production,
because "Every single time a human touches a server is an opportunity for unforced errors" (Sec. 5.4,
p. 124).

| It accumulates | Recycle it with | Source |
|---|---|---|
| Rows in the database | Purging, written in application logic | Sec. 5.4, pp. 124–126, 129 |
| Log files on disk | Rotation by size, then a copy to a staging area | Sec. 5.4, pp. 126–128 |
| Entries in an in-memory cache | A size bound and an invalidation rule | Sec. 5.4, p. 129 |

- **Purge with application logic, not with a DBA script.** A DBA can write a delete script. The DBA does
  not always know how the application behaves once the data is gone (Sec. 5.4, p. 129). Referential
  integrity and ORM collections both break (Sec. 5.4, p. 125). Measure your shortest fuse to learn how
  long you have before the accumulation harms the system (Sec. 5.4, p. 126).
- **Rotate logs by size.** Use `RollingFileAppender` in place of the default file appender. The `limit`
  and `count` properties bound the total space (Sec. 5.4, pp. 127–128). A UNIX filesystem reserves the
  last 5 to 10 percent for root, so an application gets I/O errors at 90 or 95 percent (Sec. 5.4, p. 126).
- **Ask two questions before you build a cache.** Is the space of possible keys finite? Do the cached
  items ever change? An unbounded key space needs a size limit. Anything except a finite key space with
  static items needs invalidation. A periodic time-based flush is enough nine times out of ten. Improper
  use of caching is the major cause of memory leaks. The target is a system that runs one typical
  deployment cycle with no manual disk cleanup and no nightly restart (Sec. 5.4, p. 129).

**The war story.** A team added a universal exception handler to the servlet pipeline. It logged any
exception. It was also reentrant, so an exception raised during logging was logged in turn. When the
filesystem filled, the handler went out of control. Each thread logged its own endless exception stack.
The application server consumed eight UltraSPARC III CPUs, then all available memory, then crashed the
JVM (Sec. 5.4, p. 127). A Steady State violation turned a logging safety net into a self-amplifying fire.
An automated remediation must never be reentrant on its own failure. See
`07-automated-remediation-safety.md`.

---

## 6. Fail Fast

**Sec. 5.5, pp. 131–133. Inbound.**

**Signal.** The system burns CPU time and clock time on work that it then discards. A caller waits for a
result that cannot succeed. A required dependency is already known to be down.

**Mechanism.** If the system can determine in advance that it will fail an operation, then it refuses it
at once. Nygard states the rule as "Check resource availability at the start of a transaction" (Sec. 5.5,
p. 131). He compares the practice with *mise en place*.

1. Validate the user input first. Test for null and for number format in the controller. Move richer
   validation into the domain objects or an application facade (Sec. 5.5, p. 132).
2. Determine which database connections and which integration points this request needs.
3. Acquire those connections. Read the state of the breaker around each integration point. Start the
   transaction. Preallocate memory (Sec. 5.5, p. 131).
4. If any required resource is missing, fail now. Do not start the work.

- **Report the two failure classes differently.** A system failure means that a resource is unavailable.
  An application failure means a parameter violation or an invalid state. One generic "error" can trip an
  upstream circuit breaker because a user typed bad data and pressed Reload three or four times (Sec.
  5.5, p. 132).
- **A load balancer with no working server must refuse the connection at once.** A queued connection
  request in the hope that a server appears violates Fail Fast (Sec. 5.5, p. 131).
- **Tell the caller quickly when you cannot meet the SLA.** Do not make the caller wait for an error
  message, and do not make the caller wait for its own timeout (Sec. 5.5, p. 133).

**The war story.** An older high-resolution print rendering system produced a black image of zero-valued
pixels. It did this whenever a color profile, an image, a background, or an alpha mask was missing. The
black image entered the printing pipeline and was printed, and paper, chemicals, and time were wasted. A
quality checker returned the print for diagnosis, and the expedited remake interrupted the pipeline.
Nygard's team applied Fail Fast. On arrival of a job the renderer confirmed the presence of every font,
image, background, and alpha mask. It also preallocated memory, so that a later allocation could not
fail. It reported a failure to the job control system before it spent several minutes of compute. The
software-induced remake rate dropped to zero (Sec. 5.5, pp. 131–132). The same renderer did not
preallocate disk space for the final image. That one break in Fail Fast cost the renderer several minutes. It then wrote an `IOException` to a log file (Sec. 5.5, p. 132).

---

## 7. Handshaking

**Sec. 5.6, pp. 134–135. Inbound.**

**Signal.** A caller sends work at a rate we cannot serve. We have no way to say "send less". Capacity is
unbalanced across the boundary.

**Mechanism.** Nygard defines handshaking as signaling between devices that regulates the communication
between them (Sec. 5.6, p. 134). It lets the server protect itself by throttling its own workload.

| Option | What it costs | When to use it | Source |
|---|---|---|---|
| Readiness signaling inside the protocol | Design work in a protocol you own | You define the socket protocol | Sec. 5.6, p. 135 |
| A health-check page that a load balancer polls | A crude on-or-off signal | The load balancer can remove a node | Sec. 5.6, p. 134 |
| A health-check query that the client sends first | Double the connections and requests | The added call costs much less than a call that fails | Sec. 5.6, p. 135 |
| A circuit breaker on the caller | No cooperation from the callee | The callee cannot handshake | Sec. 5.6, p. 135 |

- **Handshaking has a hard limit over HTTP.** Most clients recognize only "200 OK", "403 Authentication
  Required", and "302 Found". They treat "503 Service Unavailable" as a fatal error, although the
  specification defines it as temporary. Many clients even treat other 200-series codes as errors (Sec.
  5.6, p. 134, and footnote 11). CORBA, DCOM, and Java RMI are equally poor at signaling readiness.
- **Load-balancer-only throttling is crude.** It still breaks when every web server is too busy to serve
  another page (Sec. 5.6, pp. 134–135).
- **The client-side health check is not free.** Most of the time in a typical web service call is spent
  on the open and the close of the TCP connection. Use the extra call when its cost is much less than the
  cost of a call that fails (Sec. 5.6, p. 135).
- **A circuit breaker is the stopgap for a service that cannot handshake.** The caller makes the call and
  records whether it worked (Sec. 5.6, p. 135). This is the design rule for most vendor APIs. Build
  handshaking into any low-level protocol that you own.

**Modern.** HTTP 429 and the `Retry-After` header give a real overload signal that the 2007 text did not
have. Treat them as the protocol-level handshake when the vendor sends them. Health check faults are
catalog codes `I-24`, `I-25`, `I-26`, and `I-27`.

---

## 8. Test Harness

**Sec. 5.7, pp. 136–140. Test.**

**Signal.** No test in the suite proves what our code does when the vendor misbehaves. The integration
environment only ever returns documented responses.

**Mechanism.** Build a separate server that stands in for the remote end of the integration point and
misbehaves on purpose. It calls low-level network APIs directly (Sec. 5.7, p. 139). Every system
eventually operates outside its specification, so test the local behavior when the remote system goes
wrong (Sec. 5.7, p. 136).

| Method | What it can produce | Limit | Source |
|---|---|---|---|
| Integration test environment | The documented responses of a live dependency | Layer seven only, and not even all of it | Sec. 5.7, pp. 136, 139 |
| Mock object | Behavior that conforms to the defined interface | It cannot throw an error the interface forbids | Sec. 5.7, p. 138 |
| Test Harness | Transport, protocol, and application faults at all seven layers | It needs its own build and its own care | Sec. 5.7, pp. 138–139 |

- **A mock is not a harness.** A mock conforms to the interface. A harness runs as a separate server and
  is not obliged to conform to any interface (Sec. 5.7, p. 138).
- **Do not add a simulated-failure switch to the application.** Nygard asks who would risk the switch
  once the system reaches production (Sec. 5.7, p. 139). Have the harness log the requests instead, so
  the log names what killed the application when the application dies with no trace.
- **Supplement the other test methods. Do not replace them** (Sec. 5.7, p. 140). The fault list that a
  harness must produce has thirteen entries at Sec. 5.7, p. 137.
  `08-fault-injection-and-test-harness.md` holds that list.

**The war story.** A team wants to test against the dependency versions current at release. That wish
constrains the whole company to one new piece of software at a time. Interdependencies make the shared
environment unitary. One global environment then shadows all of production, and it needs change control
as rigorous as production, or more so (Sec. 5.7, p. 136). Nygard's own harness avoids that trap. It
selects the failure mode by port number, so one harness serves many applications with no mode switch.
Port 10200 accepts the connection and never replies. Port 10201 returns a reply copied from
`/dev/random`. Port 10202 opens the connection and drops it at once (Sec. 5.7, p. 139).

---

## 9. Decoupling Middleware

**Sec. 5.8, pp. 141–143. Architecture.**

**Signal.** A synchronous call chain crosses several systems. One slow system holds threads in every
system above it.

**Mechanism.** Choose middleware whose coupling in time and in space decides how far a shock travels.
Nygard writes that "Tightly coupled middleware amplifies shocks to the system" (Sec. 5.8, p. 141). He
calls synchronous calls particularly vicious amplifiers that facilitate cascading failures. Integration
points are the number-one cause of instability. Figure 5.4 on p. 142 orders five bands by coupling.

| Band | Coupling | Examples | Shock travels? |
|---|---|---|---|
| In-process method calls | Same time, same host, same process | C functions, Java calls, dynamic libraries | Yes |
| Interprocess communication | Same time, same host, different process | Shared memory, pipes, semaphores, Windows events | Yes |
| Remote procedure calls | Same time, different host, different process | DCE RPC, DCOM, RMI, XML-RPC, HTTP | Yes |
| Message oriented middleware | Same time, different host, different process | MQ, publish and subscribe, SMTP, SMS | No |
| Tuple spaces | Different time, different host, different process | JavaSpaces, TSpaces, GigaSpaces | No |

- **Message-oriented middleware cannot produce a cascading failure.** The requesting system does not wait
  for a reply (Sec. 5.8, p. 142). See `06-cascading-failure.md`.
- **Full decoupling reduces four antipatterns at once.** It reduces Integration Points, Cascading
  Failures, Slow Responses, and Blocked Threads. It also lets each participant change independently (Sec.
  5.8, p. 143).
- **Middleware is the exception to the cost rule.** You can apply most patterns in this chapter with
  little effect on implementation cost. Middleware products are expensive, different styles need
  different designs, and the cost of a later change of mind is very high (Sec. 5.8, pp. 142–143). Nygard
  inverts "decide at the last responsible moment" here. This decision is nearly irreversible, so make it
  early (Sec. 5.8, p. 143).

**The war story.** A card authorization built as an RPC or XML-RPC call gives a clear decision. The
application either continues to the next checkout step or returns the user to the payment methods page. A
card authorization sent as a message with no wait is harder. The system must decide what to do when the
authorization fails, or worse, stays unanswered. It needs exception queues, late responses, and callbacks
between computers and between people. The business sponsors must decide the acceptable financial risk
(Sec. 5.8, p. 142). Decoupling buys stability and moves the hard part into asynchronous design.

---

## 10. How to choose

Read the failure first. Then read the row.

| Signature you measured | Apply | Also apply |
|---|---|---|
| A thread waits inside a socket read | Use Timeouts | Circuit Breaker |
| A dependency fails at a steady rate | Circuit Breaker | Fail Fast on the inbound side |
| One consumer starves the others | Bulkheads | Handshaking |
| Latency rises at constant load over weeks | Steady State | — |
| The system does work that it discards | Fail Fast | — |
| A caller sends more than we can serve | Handshaking | Bulkheads |
| No test covers the vendor failure path | Test Harness | — |
| A shock crosses several systems | Decoupling Middleware | Circuit Breaker |

**A pattern that you did not test does not exist.** Prove each one with the Test Harness. Record the
gaps as `I-` codes from `integration-fault-catalog.md`.

---

## 11. When a pattern does not apply

State "not applicable" when it is true. That result is better than an invented finding.

- **A single in-process call needs no breaker and no network timeout.** These patterns cover calls that
  cross a boundary. A batch job with no waiting caller needs no Fail Fast.
- **A finite key space with static items needs no cache invalidation** (Sec. 5.4, p. 129).
- **Handshaking needs cooperation.** A vendor API with no readiness signal cannot handshake. Use a
  circuit breaker instead (Sec. 5.6, p. 135).
- **Decoupling Middleware is an architecture decision.** Do not raise it in a review of a small change.
  Raise it while the design is still open (Sec. 5.8, p. 143).

**Chapter 6 gives the reason to apply any of this.** Ten million page views per day for three years at
fifty assets per page gives 547,500,000,000 chances for something to go wrong. The Milky Way holds about
four hundred billion stars. Astronomically unlikely coincidences happen daily. Nygard adds that paranoia
is just good thinking (Ch. 6, p. 144).
