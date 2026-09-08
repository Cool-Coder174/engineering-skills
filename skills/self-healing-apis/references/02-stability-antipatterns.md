# The Eleven Stability Antipatterns

**Sources:** *Release It! Design and Deploy Production-Ready Software*, Michael T. Nygard
(Pragmatic Bookshelf, 2007). Chapter 4, "Stability Antipatterns", Sections 4.1 to 4.11,
pp. 44–109.

Nygard defines an antipattern as a force that appears at the root cause of more than one
system failure. Each one creates, accelerates, or multiplies a crack in the system (p. 45).

**Name the failure before you choose a defense.** A named failure has a known mechanism and a
known pattern that defeats it. An unnamed failure has neither. Two conditions let a crack
travel. Highly interactive complexity gives the operator an incomplete mental model (p. 44).
Tight coupling lets the crack cross a boundary (p. 45).

---

## Index

| # | Antipattern | Section and pages | The pattern that defeats it | Catalog codes |
|---|---|---|---|---|
| 1 | Integration Points | Sec. 4.1, pp. 46–60 | Circuit Breaker, Decoupling Middleware, Use Timeouts, Handshaking | `I-01` `I-02` `I-03` `I-32` |
| 2 | Chain Reactions | Sec. 4.2, pp. 61–64 | Bulkheads, and a fix to the defect | `I-19` `I-21` `I-46` |
| 3 | Cascading Failures | Sec. 4.3, pp. 65–67 | Circuit Breaker, Use Timeouts | `I-02` `I-05` `I-08` `I-15` |
| 4 | Users | Sec. 4.4, pp. 68–80 | A per-host Circuit Breaker, session hygiene | `I-20` `I-29` |
| 5 | Blocked Threads | Sec. 4.5, pp. 81–87 | Use Timeouts, proven concurrency libraries | `I-08` `I-20` `I-32` |
| 6 | Attacks of Self-Denial | Sec. 4.6, pp. 88–90 | Bulkheads, Fail Fast, shared-nothing design | `I-10` `I-11` `I-12` `I-42` |
| 7 | Scaling Effects | Sec. 4.7, pp. 91–95 | Shared-nothing design, lower fan-in | See `system-design` |
| 8 | Unbalanced Capacities | Sec. 4.8, pp. 96–99 | Circuit Breaker, Handshaking, Bulkheads | `I-19` `I-21` `I-22` |
| 9 | Slow Responses | Sec. 4.9, pp. 100–101 | Fail Fast, and Transparency before it | `I-05` `I-07` `I-33` |
| 10 | SLA Inversion | Sec. 4.10, pp. 102–105 | Decoupling Middleware, a breaker per dependency | `I-38` `I-39` `I-41` |
| 11 | Unbounded Result Sets | Sec. 4.11, pp. 106–109 | Handshaking, Steady State | `I-29` `I-30` |

The codes name entries in `integration-fault-catalog.md`. The patterns sit in
`03-stability-patterns.md`.

---

## 1. Integration Points (Sec. 4.1, pp. 46–60)

**Definition.** Nygard states it in one line. "Integration points are the number-one killer of
systems." (p. 46.) Every socket, process, pipe, and remote procedure call can hang.

**Mechanism.** The caller waits on a party it does not operate, and TCP hides the fault. A port
with no listener returns a reset in under 10 milliseconds on the same switch (p. 48). A full
listen queue blocks the calling thread inside the kernel for minutes (p. 49). A read with no
timeout blocks with no end, because Java blocks forever by default (p. 49). A middlebox that
drops an idle connection sends no reset packet and no ICMP message (pp. 53–56). A slow answer is
worse than no answer, because a refusal frees the thread and a slow answer holds one thread on
each side (p. 50). The vendor client library is the weak point, not the vendor server, and
blocking is its prime stability killer (pp. 57–58).

**War story — The 5 a.m. Problem (pp. 50–56).** Thirty application server instances hung inside
a five-minute window at almost exactly 5 a.m. each day. Thread dumps put every request thread
inside the Oracle JDBC library, packet capture showed almost no traffic, and the database was
healthy. The pool used last-in-first-out checkout, so one connection served the whole night and
thirty-nine sat idle past the firewall's one-hour idle timeout. The firewall deleted those
entries and told neither endpoint. The stack retransmitted into nothing, which gave a
twenty-minute write timeout on Linux 2.6 and thirty minutes on HP-UX. The fix was Oracle dead
connection detection, whose ping resets the firewall's idle timer.

**The patterns that defeat it.** Nygard names Circuit Breaker and Decoupling Middleware first
(p. 59), then adds Use Timeouts and Handshaking (p. 60). Write cynical code that expects
violations of form and function (p. 57). Build a Test Harness for each integration point (p. 59).
Read `01-integration-points.md`.

## 2. Chain Reactions (Sec. 4.2, pp. 61–64)

**Definition.** A chain reaction occurs when a defect kills one instance of a horizontal layer,
and every survivor absorbs the load it dropped (p. 61). Nygard names the usual defect. It is a
resource leak or a load-related crash (p. 61).

**Mechanism — the arithmetic.** Eight servers each carry 12.5 percent of the load. After one
dies, each survivor carries about 14.3 percent (p. 61). The absolute increase is 1.8 points,
and that server's own load rises about 15 percent (p. 61). In the two-server case the
survivor's load doubles (p. 61). Every survivor reaches the same defect sooner, so the interval
between crashes shortens.

**War story — Searching… (p. 63).** A retailer ran a dozen search engines behind a load
balancer, and health checks removed a dead engine from rotation. A memory leak in the vendor
product made engines die around noon. Because each engine had taken the same share of load all
morning, they died in an accelerating pattern. Five or six minutes separated the first crash
from the second, three or four separated the next pair, and seconds separated the last two. The
vendor patch took months, so the team ran scheduled restarts three times a day.

**The pattern that defeats it.** Bulkheads split one chain reaction into two that proceed at
different rates (p. 62). Circuit Breaker protects the callers of whichever partition fails
(p. 64). Neither is a cure. Only a fix to the underlying defect removes the chain reaction
(p. 61). Note what the war story proves about health checks. Eviction worked, and it accelerated
the deaths of the survivors (p. 63).

## 3. Cascading Failures (Sec. 4.3, pp. 65–67)

**Definition.** A cascading failure occurs when a crack in one layer triggers a crack in a
calling layer (p. 65). Nygard calls the transmission step "jumping the gap" (pp. 65–66).

**Mechanism.** The crack crosses through a resource that the calling layer holds. The text names
four carriers (pp. 65–67). A thread blocks on a call that never returns. A resource pool drains
because none of its calls return. A retry loop runs with no bound. The caller closes a
connection on any exception. Two rules follow. A safe resource pool always bounds the time a thread
waits for checkout (p. 66). Integration Points with no timeouts produce cascading failures (p. 65).
Nygard calls this class the number-one crack accelerator (p. 65).

**War story — Hammer Time (p. 67).** A lower layer held a race condition that emitted a spurious
error once in a while. Its errors carried too little detail to separate a transient fault from a
serious one. On that historical precedent the calling layer retried every quick error. Then a
failed switch started to drop database packets. The retry loop escalated until the calling layer
spent 100 percent of its CPU on calls to the lower layer and on logging the failures. A change in
the caller, not in the callee, crossed the boundary.

**The pattern that defeats it.** Circuit Breaker prevents the call to the sick integration point.
Use Timeouts lets the caller return from a call it already made (p. 67). Read
`06-cascading-failure.md`.

## 4. Users (Sec. 4.4, pp. 68–80)

**Definition.** Users cost memory, time, and money. Nygard splits the risk into four classes.
They are traffic (p. 68), users who are expensive to serve (p. 71), unwanted users (p. 72), and
malicious users (p. 78).

**Mechanism.** Capacity scales with the hardware and the bandwidth you bought, not with the user
count you attracted (p. 68). Each session holds memory through the dead time after the last
request (p. 69). Under memory pressure the logging framework cannot allocate, so nothing gets
logged (p. 69). A user who buys is expensive, because checkout touches card authorization, tax,
address standardization, inventory, and shipping (pp. 71–72). Robots and scrapers often ignore
session cookies, so each request creates a new session (pp. 74, 76).
Three rules carry numbers. Keep the session small and requery for each page (p. 69). Wrap a
large object in a `SoftReference`, and handle a null payload (pp. 70–71). Test at 4, 6, or 10
percent conversion when your baseline is 2 percent (p. 72).

**War story — the Navy base session flood (pp. 73–74).** A badly configured proxy server started to
re-request one user's last URL fifteen minutes after his last request. The rate began at one
request every thirty seconds and reached five each second. Each request carried his identifying
cookie and no session cookie, so each one created a new session. A developer had added an
interceptor that updated the "last login" time whenever a profile loaded, so about 100,000
transactions then contended for one row. One of them hung while it waited on a different resource
pool. That thread blocked every other transaction on the row, consumed every request-handling
thread, and stopped the site.

**The pattern that defeats it.** A specialized per-host Circuit Breaker limits the damage that
any one host does (p. 79). Expiring blocks at the firewall are a form of Circuit Breaker (p. 78).
Do not copy a threshold from vendor documentation. At fifteen connections per minute per source
address, every AJAX application is a denial of service (p. 79). A deliberate flood belongs to
`security-engineering`. Most machine users of an API today are SDK clients, and the mechanism
holds. **Modern**

## 5. Blocked Threads (Sec. 4.5, pp. 81–87)

**Definition.** Every thread that can process a transaction waits on an outcome that never
arrives (p. 81). The process runs, and it completes no work. Nygard rates it first. Blocked
Threads is the proximate cause of most failures (p. 87).

**Mechanism.** A thread blocks in one of four places. They are a critical section, a resource
pool, a cache or object registry, and a call to an external system (p. 81). A blocked thread is
often found near an integration point (p. 87). The block is invisible at the call site, because
neither the calling code nor the interface declares synchronization (p. 83). You cannot test
hangs out of the system, for four stated reasons (pp. 81–82). Error conditions create too many
permutations. Unexpected interactions break safe code. Timing decides the outcome. Developers
never send 10,000 concurrent requests at their own code.

**Rules.** Use concurrency primitives from a proven library, and never write your own connection
pool (pp. 82–83). Treat the `synchronized` keyword on a method as an alarm (p. 84). Scrutinize
every resource pool, because a deadlock and bad exception handling lose connections forever
(p. 87). You cannot prove that your code holds no deadlock, and you can make sure that no
deadlock lasts forever (p. 87).

**War story — RemoteAvailabilityCache (pp. 85–86).** A retail site needed in-store item
availability from a remote inventory system, and up to 4,000 concurrent requests could arrive. A
developer built a textbook read-through cache. He extended `GlobalObjectCache` and overrode
`create()` to make the remote call, with timestamps for staleness. The design was functionally
correct. The inherited `get()` was synchronized. The undersized inventory back end then failed
under front-end load, and one thread inside `create()` waited for an answer that never came. That
thread blocked every other caller, and the site failed over an in-store pickup check. No one
designed this failure mode in, and no one designed it out.

**The pattern that defeats it.** Use Timeouts on every wait, including the version of `wait()` that
takes a timeout (p. 87). Add external monitoring. A mock client outside the data center runs
synthetic transactions, and its failure is the alarm whether or not the process runs (p. 82). Probe
a third-party library on purpose. Use a Test Harness that holds every connection, and watch for a
throughput drop at its hidden pool size (p. 86). When the library exposes no timeout, hand the call
to a worker thread outside it and abandon the call at the deadline (p. 86).

## 6. Attacks of Self-Denial (Sec. 4.6, pp. 88–90)

**Definition.** Any situation in which the system, or the extended system that includes the
people, conspires against itself (p. 88).

**Mechanism.** Your own organization creates the flash mob. Any special offer meant for 10,000
users attracts millions, and a redemption cap does not save you (pp. 88, 90). Three gates apply to
mass email (p. 90). Send no deep links, build static landing zone pages, and keep session
identifiers out of URLs. The machine form is different. One rogue server damages every other server
through a shared resource (p. 89). One ATG lock manager handles distributed locks. A programming
error on a popular item can then serialize thousands of request-handling threads on hundreds of
servers behind one write lock (p. 89).

**War story — the Xbox 360 preorder email (p. 88).** A large electronics retailer emailed a
preorder promotion to a select group at a time when demand clearly exceeded United States supply.
The email named the exact date and time the preorder would open. It also carried a deep link that
bypassed Akamai, so every image and stylesheet came from the origin servers. One minute before the
appointed time the whole site lit up, and it was gone in sixty seconds. In a second case, Amazon
sold 1,000 units at $100 in five minutes, while nothing else sold, because visitors pressed Reload
on one undedicated cluster.

**The pattern that defeats it.** Reserve a part of the infrastructure for the surge. This works
only when the extraordinary traffic targets one part of the system (p. 89). Use Fail Fast when
the dedicated servers stop answering, so upstream connections do not wait for a useless answer
(p. 90).
For the machine form Nygard gives four options in order (p. 89). Build a shared-nothing
architecture. Apply decoupling middleware. Make the shared resource horizontally scalable. Design
a fallback mode, such as optimistic locking when the pessimistic lock manager is unavailable. Our
own version of this antipattern is the retry storm. Read `05-overload-and-load-shedding.md`.

## 7. Scaling Effects (Sec. 4.7, pp. 91–95)

**Definition.** A relationship that is safe at a one-to-one ratio becomes fatal when one side of
it grows (p. 91). Development looks like one server, QA looks like two, and production looks
like many. Nygard uses the square-cube law as the analogy (p. 91).

**Mechanism.** Point-to-point communication is the first case. Each instance talks directly to
every other instance, so the connection count grows as the square of the instance count (p. 92).
Two servers are acceptable, as long as the code does not block when the other server dies
(p. 92). At tens of servers you will probably need one-to-many communication (p. 95). A shared
resource is the second case. A redundant, nonexclusive resource is safe, because you can add
more of it (p. 93). The dangerous case is a resource held exclusively for one unit of work
(p. 93). The contention probability scales with the transaction count and the client count in
that layer (p. 94). The saturation order is fixed. The resource saturates, a backlog forms, it
exceeds the listen queue, and transactions fail (p. 94).

**No war story.** Section 4.7 gives worked examples instead.

**The pattern that defeats it.** Shared-nothing design is the counterweight, because capacity
then scales close to linearly with the server count, at the cost of failover (p. 94). This
defect cannot be tested out. It must be designed out, because you cannot build a test farm the
size of production (p. 92). Replace point-to-point communication with UDP broadcast, multicast,
publish and subscribe messaging, or message queues, and choose the simplest thing that will work
(pp. 92–93). Where shared-nothing design is impractical, reduce the fan-in (p. 94). Stress test
the shared resource, and prove its clients keep working when it is slow (p. 95). No `I-` entry
owns architecture selection. Send that decision to `system-design`.

## 8. Unbalanced Capacities (Sec. 4.8, pp. 96–99)

**Definition.** The front end always has the ability to overwhelm the back end, because their
capacities are not balanced (p. 98). Nygard calls it a special case of Scaling Effects, where
one side of a relationship scales far more than the other (p. 99).

**Mechanism.** Over a short period your hardware capacity is fixed, and adding capacity takes
weeks, or days in a crisis (pp. 96–97). Figure 4.13 gives the concrete ratio. The front end holds
20 hosts, 75 instances, and 3,000 threads. The back end holds 6 hosts, 6 instances, and 450 threads
(p. 97). A change in the traffic mix moves a larger fraction of those 3,000 threads onto the back
end. Its load then rises by 2, 4, or 10 times (p. 97). QA proves nothing, because QA is one-to-one
while production runs ten to one or worse (pp. 98–99). Three thousand threads calling into
seventy-five threads is not in the ballpark (p. 99).

**War story — the free installation promotion (pp. 97–98).** A retail website normally sent a tiny
fraction of its 3,000 request-handling threads into a scheduling system that held 450 threads
across 6 hosts. Marketing then offered free installation of any big-ticket appliance for one day
only. The fraction grew by two, four, or ten times, and the scheduling system was overwhelmed.
Building the scheduling system to the website's size would leave it 99 percent idle except for one
day in five years. Equal capacity was never the answer.

**The pattern that defeats it.** Apply Circuit Breaker on the front end, so it relieves the
pressure when answers get slow. Apply Handshaking on the back end, so it can tell the front end
to reduce its rate. Add Bulkheads on the back end to reserve capacity for other transaction
types (p. 98). Use capacity modeling first, then test abnormal load (p. 99). The back-end test
is exact. Take the highest call volume the front end can produce, double it, and direct all of
it at your most expensive transaction (p. 99). It passes when the system slows, may Fail Fast,
and recovers. It fails on crashes, hung threads, empty responses, or nonsense replies (p. 99).
Four triggers change the workload that reaches an unchanged back end. They are marketing
campaigns, publicity, new front-end releases, and links on funnel sites (p. 99).

## 9. Slow Responses (Sec. 4.9, pp. 100–101)

**Definition.** The system generates a slow answer instead of a fast failure. Nygard ranks it
plainly. A slow response is worse than a refused connection or an error, and the ranking matters
most for middle-layer services (p. 100).

**Mechanism.** A quick failure lets the calling system finish the transaction quickly. A slow
answer holds resources in the calling system and in the called system (p. 100). Slow responses
move upward from layer to layer as a gradual cascading failure (p. 100). Upstream systems break
when the response time exceeds their own timeout (p. 101), so timeout budgets must agree across
layers. The named causes are five (p. 100). Excessive demand leaves no slack. A memory leak shows
as high CPU that is garbage collection rather than work. Network congestion is real across a WAN.
A TCP stall appears when a read routine does not loop until the receive buffer drains. Connection
pool contention produces the same symptom, and so does an unbounded result set (p. 109). The
cycle reinforces itself, and users who wait press Reload (p. 101).

**No war story.** Section 4.9 has none. Nygard demonstrates the mechanism inside the
RemoteAvailabilityCache case, in Section 5 of this file (pp. 85–86).

**The pattern that defeats it, with the book's numbers.** Fail Fast, and Transparency before it,
because the system must track its own responsiveness (pp. 100–101). Nygard's worked example uses
a 100 millisecond requirement and a moving average over the last twenty transactions. When that
average exceeds 100 milliseconds, the system can start to refuse requests (p. 100). Refuse at
the application layer with an error inside the defined protocol, or at the connection layer by
refusing new socket connections (p. 100). The refusal must be documented, and the callers must
expect it (p. 100). The weakest acceptable trigger is the average response time above the
caller's timeout (p. 101). Those numbers are the book's example. Yours are a local decision, and
they come from your measured latency distribution.

## 10. SLA Inversion (Sec. 4.10, pp. 102–105)

**Definition.** A system that must meet a high-availability SLA depends on systems of lower
availability (p. 104).

**Mechanism — joint probability.** A single failure in one dependency fails you, so the
availabilities multiply (p. 103). Unless every dependency is engineered for the SLA you must
provide, the best you can do is the SLA of your worst provider (p. 103). Five external services
at 99.9 percent each cap you at 99.5 percent (p. 103). For scale, 99.99 percent allows slightly
more than four minutes of downtime per month (p. 102). A perfectly decoupled system is bounded
only by its own internal failure probability, and most systems sit between the bounds (p. 104).
Every dependency exposes three layers that fail independently. They are transport availability,
the naming service (DNS), and the application protocol (p. 103).

**War story — Project Frammitz (pp. 102–105).** A mission-critical new website carried redundancy
at every level, including power, network, storage, server hardware, and applications, plus a
shared-nothing architecture. Its commitment was 99.99 percent. Its dependency list ran to
settlement, fulfillment, inventory, fraud detection, a channel partner, geocoding, and card
authorization. The recorded SLAs ran from 99.999 percent down to 98.5 percent, and two
dependencies offered no SLA at all. Nygard's verdict is that Frammitz can meet its number only
through luck. The engineering was real, and it did not move the ceiling.

**The pattern that defeats it.** Decouple from the lower-SLA system, keep the application working
without it, and degrade (p. 104). Decoupling middleware is one approach. At minimum, employ a
circuit breaker per dependency (p. 104). Then restate the agreement. Give your maximum SLA to
features that depend on no external party. Give each remaining feature the SLA its third party
offers, reduced by your own failure probability (p. 105). Inventory every dependency, including
corporate DNS, SMTP, message brokers, and the enterprise SAN (p. 105). Service levels only decrease
when you call third parties (pp. 104–105). Read `09-vendor-slas-and-degradation.md`.

## 11. Unbounded Result Sets (Sec. 4.11, pp. 106–109)

**Definition.** The code sends a query and then processes every row it receives, with no limit
that the caller set (p. 106). Nygard names the class precisely. An unbounded result set occurs
when the caller lets the other system dictate the terms, and it is a failure in handshaking
(p. 108).

**Mechanism.** Development and test datasets are small, so developers never meet the outcome
(p. 108). After a year in production, even a traversal such as "fetch customer's orders" can
return a huge result set (p. 108). ORM tools limit a query, and they usually do not limit an
association traversal (p. 108). Suspect any relationship that accumulates unlimited children,
such as orders to order lines or profiles to site visits, and treat audit-trail entities as
suspect (p. 108). The same failure appears in web services, RMI, DCOM, and AJAX (pp. 108–109).

**War story — Black Monday (pp. 106–108).** With no warning, every one of more than 100
load-balanced instances of a commerce server rose to 100 percent CPU. Each one crashed with a
HotSpot memory error three or four minutes later. Some instances crashed during cache warm-up
before they accepted a request, which eliminated incoming traffic as the cause. Heap available
headed to zero, the crash mode implied native code, and a stack dump named the Type 2 JDBC driver.
A query trace named the last statement. It read a JMS message table that should never hold more
than 1,000 rows and held more than ten million. Each instance selected every row with `select for
update`, exhausted memory, and crashed. The rollback released the lock, and the next instance ran
the same query.

**Rules.** In any API or protocol, the caller should always state how much of an answer it is
prepared to accept (p. 108). TCP's window field is the precedent, and a search API that takes a
count and an offset is the applied form (p. 108). The only sensible numbers are zero, one, and
lots, so any query that does not select exactly one row can return too many (p. 109). Do not
rely on the data producers (p. 109). Test with production-sized data volumes (p. 109). When you
cannot change the query, leave the processing loop at the maximum row count, and accept that it
wastes database capacity (p. 108).

**The limit recipes Nygard gives (p. 108).** No standard SQL syntax expresses a result-set limit.
The value 15 is the book's example, and your own limit is a local decision.

| Database | Syntax |
|---|---|
| Microsoft SQL Server | `SELECT TOP 15 colspec FROM tablespec` |
| Oracle, 8i and later | `SELECT colspec FROM tablespec WHERE rownum <= 15` |
| MySQL and PostgreSQL | `SELECT colspec FROM tablespec LIMIT 15` |

**The pattern that defeats it.** Handshaking, because the caller must declare its capacity
(p. 108). Steady State also applies, because an unbounded result set can follow from a violation
of steady state (p. 109).

## How the antipatterns connect

Nygard's chapter is a graph. Read this table when one name does not explain the whole incident.

| This antipattern | Leads to | Source |
|---|---|---|
| Integration Points with no timeouts | Cascading Failures | Sec. 4.3, p. 65 |
| Blocked Threads | Chain Reactions, then Cascading Failures | Sec. 4.5, p. 87 |
| Chain Reactions in one layer | Cascading Failure in the calling layer | Sec. 4.2, p. 62 |
| Slow Responses | Cascading Failures | Sec. 4.9, p. 101 |
| Unbounded Result Sets | Slow Responses | Sec. 4.11, p. 109 |
| Self-Denial through a shared resource | Blocked Threads across the farm | Sec. 4.6, p. 89 |

**One incident usually carries several names.** The RemoteAvailabilityCache case combines Blocked
Threads, Unbalanced Capacities, and Integration Points with no timeouts (p. 85). Report each one.

## Where to read next

| File | Read it when |
|---|---|
| `01-integration-points.md` | You add or review any outbound call |
| `03-stability-patterns.md` | You choose the defense for a name in this file |
| `04-fault-localization.md` | You hold a symptom and must prove where the fault is |
| `05-overload-and-load-shedding.md` | Retry storms, budgets, criticality, deadlines |
| `06-cascading-failure.md` | The failure crossed a layer boundary |
| `08-fault-injection-and-test-harness.md` | You must prove the failure path works |
| `09-vendor-slas-and-degradation.md` | You promise a number, or you choose a degraded mode |
| `integration-fault-catalog.md` | Every review, and every incident |
