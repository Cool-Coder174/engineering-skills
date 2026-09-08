# Capacity Defect Catalog

**Sources:** *Release It! Design and Deploy Production-Ready Software* — Michael T. Nygard
(Pragmatic Bookshelf, 2007). Ch. 7 to Ch. 10, p. 147–217, Ch. 13 and Ch. 14, p. 232–246.
*Site Reliability Engineering: How Google Runs Production Systems* — Beyer, Jones, Petoff,
Murphy (O'Reilly, 2016). Ch. 3 to Ch. 5, p. 58–81, Ch. 17, p. 228, Ch. 18 to Ch. 21, p. 248–312.

A catalog of named capacity defects. Each one is detectable in a design, a difference, a
configuration file, or a plan, before the system reaches its limit. Each entry has three parts:

- **Signature** — what the code, the config, or the design looks like. Search for this text.
- **Consequence** — what goes wrong, and when.
- **Fix** — the required remedy.

**How to use it.** `code-review` runs the applicable groups against a difference. `verify` runs
them against an implemented phase. `planner` and `detail-planning` run them against a design.
Report only the defects that apply. "Not applicable. This change adds no call, no query, and no
per-request work." is a valid result, and it is better than an invented finding.

**Severity.** 🔴 causes data loss, corruption, or an unrecoverable state. It blocks the merge.
🟡 causes an outage or a wrong result under load. 🔵 is a risk to operation or to maintenance.

**Citation form.** A citation such as `Sec. 9.1, p. 176` names *Release It!*. A citation that
starts with `SRE` names *Site Reliability Engineering*. A claim that both books predate carries
the tag **Modern**. Neither book states a **Modern** claim.

---

## A. The capacity number and the constraint

### 🟡 C-01 — No constraint is named
**Signature:** a design, a plan, or a ticket states a capacity number and names no resource.
Search for "will scale", "should handle", and "no problem at that volume".
**Consequence:** the team improves a metric that is not the constraint and gains nothing.
Exactly one constraint determines the capacity of a system (Sec. 8.2, p. 163).
**Fix:** run the constraint procedure in `../SKILL.md`, Section 3. Name one driving variable,
one following variable, and the correlation.

### 🟡 C-02 — Capacity projected from a nonconstraint metric
**Signature:** arithmetic of this shape. "We handle 10,000 users at 50 percent CPU, so we
handle 20,000 users." A plan whose only input is average CPU.
**Consequence:** the projection is wrong, because the constraint is elsewhere. Nygard names
this reasoning as the fallacy. Any nonconstraint metric is useless (Sec. 8.2, p. 163).
**Fix:** project from the constraint only. Improving a nonconstraint metric does not improve
capacity (Sec. 8.6, p. 174).

### 🔵 C-03 — A capacity number with no workload and no response-time bound
**Signature:** "the system supports 5,000 transactions per second". No transaction mix and no
acceptable response time.
**Consequence:** nobody can test the number and nobody can refute it. Capacity is the maximum
throughput for a given workload at an acceptable response time (Sec. 8.1, p. 162).
**Fix:** state the mix and the bound per operation. Nygard gives two seconds for retail and
five hundred milliseconds for an availability search (Sec. 8.1, p. 162).

### 🟡 C-04 — Capacity read from a mean
**Signature:** a dashboard, an alert, or a capacity report whose latency panel shows `avg()`.
A load-test report that gives one average response time.
**Consequence:** the mean hides the tail. SRE describes a typical 50 ms request where 5 percent
of requests are 20 times slower, and where the average does not move (Ch. 4, p. 67–68).
**Fix:** report p50, p95, p99, and p999 (SRE Ch. 4, p. 68). `data-systems-design` owns the
aggregation mechanism.

### 🔵 C-41 — Capacity measured one time only
**Signature:** one capacity number, dated at the last launch. No capacity measurement in the
release process, and no comparison of one test against two builds.
**Consequence:** a release changes the workload and the number becomes wrong. Monitor capacity
continuously, because each release can affect scalability and performance (Sec. 8.6, p. 174).
**Fix:** measure the constraint metric on each release. An outage from a capacity regression
spends the error budget (SRE Ch. 3, p. 59–60). `reverse-branching` owns the revert.

---

## B. The load test

### 🟡 C-05 — The load script follows the happy path only
**Signature:** every virtual user requests a URL, waits for the response, and then requests a
link from that response. Every script accepts cookies.
**Consequence:** the test passes and production fails. Nygard's site passed a three-month
campaign of such scripts and crashed thirty minutes after the launch (Sec. 7.4, p. 155–157).
**Fix:** add the noise that `../SKILL.md`, Section 4 lists. Nygard states the rule. Do not
follow the happy path only (Sec. 7.5, p. 157).

### 🟡 C-06 — The test contains no noise traffic
**Signature:** a load plan that lists page flows and conversion rates, and lists no spiders,
no cookieless clients, no URL floods, and no 404 traffic.
**Consequence:** noise consumes capacity and can stop the site. One spider created ten sessions
per second, each resident for thirty minutes. Dead URLs drew 40 percent of visits (Sec. 7.4,
p. 156).
**Fix:** add a cookieless client, a 404 generator, a URL flood, and a scraper profile. Serve
404s without the application server (Sec. 7.6, p. 159).

### 🔵 C-42 — The load generators themselves are never verified
**Signature:** a ceiling reported from one test farm. No check of the generator hosts, and no
check of the network path to the system.
**Consequence:** the ceiling can belong to the test harness. Attackers took Nygard's generator
farm, and a carrier engineer limited the subnet that produced 80 percent of the load
(Sec. 7.3, p. 152–155).
**Fix:** measure the generators as you measure the system. Confirm the network path before you
accept a ceiling.

### 🟡 C-07 — The load generator waits for the response — **Modern**
**Signature:** a test tool configured with a fixed number of virtual users and no arrival rate.
The tool sends the next request after the previous response arrives.
**Consequence:** the generator stops sending load exactly when the system slows. The queue never
grows and the report flatters the system. This effect is named coordinated omission.
**Fix:** drive the test at a fixed arrival rate, independent of the response time.
`data-systems-design` owns the rule. This skill owns the test configuration.

### 🟡 C-08 — The test environment does not match the production topology
**Signature:** one instance in test against a cluster in production. Two applications that
share a host in test only. No firewall and no load balancer in the test path.
**Consequence:** the result does not transfer. The usual cause of a failed deployment is a
topology mismatch, not a configuration mismatch (Sec. 14.1, p. 241). One instance also hides
multicast invalidation (Sec. 14.1, p. 242).
**Fix:** apply the four rules in Sec. 14.1, p. 242–243. Separate the hosts. Run more than one
instance. Keep the firewalls. Buy the same load balancer product line.

### 🟡 C-09 — No test past the limit
**Signature:** a test plan whose highest load step equals the target. No run continues until a
component fails.
**Consequence:** nobody knows the failure mode. Components do not degrade gracefully past a
point, and instead fail catastrophically (SRE Ch. 17, p. 228). Nygard's site needed nearly an
hour to serve pages again (Sec. 7.6, p. 158).
**Fix:** run a stress test until a component fails. Record the knee, the first component that
fails, and the recovery time. Set the operating limit below the knee.

---

## C. Pooled resources and request threads

### 🟡 C-10 — The pool is smaller than the request thread count
**Signature:** a request thread count and a connection pool size in one configuration file.
The pool holds the smaller number. Example: 30 threads and 4 connections.
**Consequence:** contention stays at zero until the thread count passes the resource count, and
throughput then flattens at the knee. Four connections against thirty requests waste more than
80 percent of CPU time (Sec. 9.1, p. 176).
**Fix:** make the resource pool size equal to the number of request threads (Sec. 9.1, p. 176).
Check C-14 before you apply this.

### 🔴 C-11 — A thread waits for a pooled resource without a limit
**Signature:** a pool configuration with no `maxWait` and no `<blocking-timeout-millis>`. A
`getConnection()` call with no timeout argument.
**Consequence:** a wait without a limit at resource exhaustion guarantees a stability problem
(Sec. 9.1, p. 178). New threads enter the broken path, and the failure has no floor.
**Fix:** configure a bounded wait. Nygard names `maxWait` in Jakarta Commons `BasicDataSource`
and `<blocking-timeout-millis>` in JBoss (Sec. 9.1, p. 178). `data-systems-design` owns the
general timeout rule as H-15. This entry owns the pool case.

### 🟡 C-12 — The timeout exists and the caller has no path for it
**Signature:** a bounded pool wait, and a call site that handles neither a null return nor the
timeout exception.
**Consequence:** the pool returns null or throws, and the request fails in an undefined way.
The application code must be prepared for this (Sec. 9.1, p. 178).
**Fix:** define the caller behavior. Return a degraded response, or fail with a defined code.
The caller knows what to do when it gets no connection (Sec. 10.1, p. 207).

### 🔵 C-13 — The pool has no metrics
**Signature:** no counter for blocked callers, no high-water mark, and no create-and-destroy
count. No collector reads the pool.
**Consequence:** contention stays invisible until it becomes an outage. Nygard requires the
frequency of blocking, the high-water mark, and the count of resources created and destroyed
(Sec. 9.1, p. 179).
**Fix:** expose the three metrics and read them on a schedule. JMX exposes some of them, and
you must poll them yourself (Sec. 9.1, p. 179).

### 🟡 C-14 — The fleet-wide connection total is never computed
**Signature:** a per-instance pool size, and no arithmetic against the instance count and the
machine count. No statement of the database connection limit.
**Consequence:** 20 machines with 5 instances at 50 connections is 5,000 connections, which is
5 GB of database server memory (Sec. 9.1, p. 177). Under failover, one node serves every query
and every connection (Sec. 9.1, p. 179).
**Fix:** compute the total. Compare it against the server limit in the degraded topology, not
in the healthy one.

### 🟡 C-43 — A failed connection returns to the pool
**Signature:** a pool with no validation query and no eviction rule. A code path that returns a
connection to the pool after an error on it.
**Consequence:** a bad connection returns fast, so the pool offers it more often than a healthy
one. One bad connection in ten causes more than 10 percent of requests to fail (Sec. 10.1,
p. 206).
**Fix:** validate a connection at checkout. Destroy a connection after an error on it. Watch
the create-and-destroy count from C-13.

---

## D. Sessions and client state

### 🟡 C-15 — The session timeout is the default
**Signature:** no `session-timeout` element, or a value of 30 minutes. The framework default in
the deployment descriptor.
**Consequence:** sessions stay in memory long after the user is gone, and the memory cost is
proportional to their tenure. Nygard calls the 30-minute default overkill (Sec. 9.4, p. 185).
**Fix:** set the timeout to one standard deviation past the average think time. Nygard measured
about 10 minutes for retail, 5 for a media gateway, and 20 for travel (Sec. 9.4, p. 185).

### 🟡 C-16 — Whole objects live in the session
**Signature:** `session.setAttribute` with a cart, a result set, or a domain object. A session
store whose average entry is measured in kilobytes.
**Consequence:** session replication becomes unaffordable and the team disables it. Nygard's
users in checkout on a lost instance returned to the cart page and left (Sec. 7.6, p. 159–160).
**Fix:** keep keys, not whole objects, and use soft references for an object (Sec. 9.4, p. 186).
Acceptable content is a user ID, a cart ID, and a search key (Sec. 7.6, p. 160).

### 🟡 C-17 — Every request creates a session
**Signature:** a web server rule that routes every `.html` path to the application server. A
404 handler inside the application. An asynchronous request with no session identifier.
**Consequence:** a client that never returns a cookie creates one session per request (Sec. 7.4,
p. 156). An asynchronous request with no identifier creates a wasted session (Sec. 9.3, p. 184).
**Fix:** serve 404s and unidentified requests from static content (Sec. 7.6, p. 159). Carry the
session identifier on every asynchronous request (Sec. 9.3, p. 184).

### 🟡 C-18 — Client state travels on every request
**Signature:** a serialized object in a cookie. A token whose payload holds a profile, a
permission list, or a cart. A cookie larger than a few hundred bytes.
**Consequence:** the payload crosses the wire two times, and sometimes four times, on the user's
limited upstream bandwidth. A serialized form outlives the code (Sec. 9.10, p. 201–202).
**Fix:** cookies carry identifiers only, and the state stays on the server. Nygard's rule is
about 100 bytes (Sec. 9.10, p. 202–203). **Modern:** a signed token obeys the same rule.

---

## E. Per-request waste and the multiplier

### 🟡 C-19 — Per-request work on data that changes rarely
**Signature:** a render, a transform, a filter, or a lookup inside the request path, over
content that a batch job publishes one time a day.
**Consequence:** the system pays the cost a million times a day for a benefit gained one time
a week (Sec. 10.5, p. 217). The Profanity Masker created 10 MB of garbage per request (p. 211).
**Fix:** precompute when the source data changes, and punch out a hole for the personalized
fragment (Sec. 10.3, p. 210–212). Do the most work when nobody waits (Sec. 8.6, p. 174).

### 🔵 C-20 — The response size is never measured
**Signature:** no byte-size check in the test report. No compression on a text response. A
template that emits blank lines around tags that produce nothing.
**Consequence:** the request count multiplies every excess byte. One page above 600 KB carried
200 KB of newlines, which cost web server memory and more than $15,000 a year in bandwidth
(Sec. 9.5, p. 188).
**Fix:** measure the byte size of each response in the test, and remove the waste at its source
(Sec. 9.5, p. 188). **Modern:** the rule also covers an uncompressed JSON response.

### 🟡 C-21 — A chatty remote call, or the 1+N pattern
**Signature:** one call to fetch a collection, and one or more calls per member. A remote
interface with many small methods. A loop that calls a service.
**Consequence:** a remote call takes at least 1,000 times as long as a local call (Sec. 9.9,
p. 199). A blocked thread still holds memory, a CPU slice, and a database connection.
**Fix:** add a coarse method that returns summary objects with exactly the fields the caller
needs (Sec. 9.9, p. 200). `self-healing-apis` owns Circuit Breaker (Pattern 5.2, p. 115).

### 🟡 C-22 — The poll interval is shorter than the think time
**Signature:** a timer that fires a request every 250 milliseconds. A client that polls for
status on a short fixed interval. A page that fires a request on every keystroke.
**Consequence:** the interval between requests falls from five to ten seconds to one to three
seconds. The request count rises against the same user population (Sec. 9.3, p. 182).
**Fix:** send a request when the input changes, or 500 milliseconds after the user stops typing
(Sec. 9.3, p. 183–184). **Modern:** a status endpoint obeys the same arithmetic.

### 🟡 C-44 — The client repeats a slow request
**Signature:** a page or a call with no bound on its response time. No idempotency key on the
write path. A rule that serializes requests by source address.
**Consequence:** a user presses Reload after about ten seconds, and nobody stops the first
request (Sec. 9.6, p. 191). The second request can block on the first one, and the two can
deadlock (Sec. 9.6, p. 192).
**Fix:** make the transaction safe for one user who runs it several times. Do not serialize on
source address, because a proxy hides many users behind one address (Sec. 9.6, p. 192).

---

## F. The database

### 🟡 C-23 — Handcrafted SQL that joins on unindexed columns
**Signature:** a raw SQL string in application code, outside the ORM. A join across many tables.
A `WHERE` clause built by string concatenation.
**Consequence:** the access pattern is unpredictable, so tuning cannot serve it and can harm the
rest of the application. Nygard reports one query with about forty table scans in its plan
(Sec. 9.7, p. 193–194).
**Fix:** prefer the predictable SQL that the ORM generates. Try an index, a hint, or a view
first. Apply the laugh test before production (Sec. 9.7, p. 194–195).

### 🟡 C-24 — An ORM association target is not indexed
**Signature:** a mapping file or an entity annotation that names a foreign column, and a
migration that creates no index on that column.
**Consequence:** the mapping file triggers no database review, so a table scan reaches
production. A scan that is invisible on development data becomes a wait of minutes after a year
(Sec. 9.8, p. 196).
**Fix:** index every column that is the target of an association in the ORM mapping. The first
iteration of indexes is the developer's responsibility (Sec. 9.8, p. 196–198).

### 🟡 C-25 — The query was measured on development-sized data
**Signature:** a performance claim in a pull request, against a test database that holds
hundreds of rows. A seed script that creates a toy data set.
**Consequence:** the production query plan differs, so the gain disappears (Sec. 9.7, p. 195).
On a small table a scan can even be the fastest plan, which hides the defect.
**Fix:** run against production-sized data, scrubbed by a random scramble of the private
characters. Where no data exists, build a data generator (Sec. 9.7, p. 194).

### 🟡 C-26 — Reports run against the transactional database
**Signature:** an analytics query, an export job, or an admin dashboard whose connection string
names the production transactional database.
**Consequence:** reporting competes with transactions for the constrained resource. An OLTP
schema is optimized for fast inserts and is bad for reports and ad hoc queries (Sec. 9.8,
p. 198).
**Fix:** do not mix transactions and reporting. Serve reports from a star schema. Restrict a
user-facing report to the last ninety days or six months (Sec. 9.8, p. 198).

### 🔵 C-27 — No growth rate and no purge plan
**Signature:** a new table with no retention rule, no partition key, and no archive job. An
audit table or an event table with unbounded growth.
**Consequence:** one release changed an audit log from 1 GB per year to 1 GB per day. The table
then spread across extents until disk I/O dominated the response time (Sec. 9.8, p. 197).
**Fix:** state the row size, the rows per day, and the retention period. A rigorous regimen of
data purging is vital to long-term stability (Sec. 9.8, p. 197–198).

---

## G. Cache, precomputed content, and memory

### 🟡 C-28 — A cache with no maximum size
**Signature:** a map or a dictionary used as a cache, with no capacity argument and no eviction
policy. A cache library constructed with defaults.
**Consequence:** the cache consumes the memory that requests need. The collector works harder,
and the cache itself causes the slowdown (Sec. 10.2, p. 208).
**Fix:** make the maximum memory of every application-level cache configurable. In Java, hold
cached items through soft references (Sec. 10.2, p. 208).

### 🔵 C-29 — A cache with no measured hit rate
**Signature:** a cache with no hit counter and no miss counter. No panel on a dashboard.
**Consequence:** a cache with a very low hit rate buys nothing and can be slower than no cache.
An object used one time in the life of a server does not help (Sec. 10.2, p. 208).
**Fix:** count hits and misses per cached class and report the rate. Remove a cache that does
not earn its memory. Do not cache an object that is cheap to create (Sec. 10.2, p. 208).

### 🟡 C-30 — A cache with no invalidation rule, or an unlimited flush
**Signature:** a cache with a time-to-live only, and no removal when the source data changes. A
flush endpoint or an event handler that any change can trigger.
**Consequence:** stale data reaches users. A flush that starts too often produces an attack of
self-denial. At hundreds of servers, point-to-point invalidation stops working (Sec. 10.2,
p. 209).
**Fix:** remove an item when its source data changes. Limit the flush rate. Prevent a
simultaneous reload on every server (Sec. 10.2, p. 209). `data-systems-design` owns correctness.

### 🟡 C-31 — The collector is never tuned, and class loading has no bound
**Signature:** no garbage collection log and no `-verbosegc` in the start script. A heap setting
copied from the previous release. A template count that grows with the content.
**Consequence:** an untuned application at production volume spends about 10 percent of its
runtime on garbage collection (Sec. 10.4, p. 214). An unbounded template count gives an
unbounded memory region (Sec. 9.2, p. 180).
**Fix:** size the heap and adjust the generation ratios in production at production traffic.
Target 2 percent or less, and retune after each major release (Sec. 10.4, p. 214–217).

### 🟡 C-45 — The cache is cold when the instance accepts traffic
**Signature:** an in-memory cache with no warm phase. A deployment that sends traffic to a new
instance at once. A fragment cache larger than the free memory.
**Consequence:** the first request against cold caches can wait minutes for a single page
(Sec. 10.3, p. 212–213). A restarted task also needs more resources (SRE Ch. 20, p. 293).
**Fix:** precompute the content to storage instead of an in-memory fragment cache (Sec. 10.3,
p. 212). Hold the task in lame duck state and prewarm it (SRE Ch. 20, p. 293).

---

## H. Load distribution and utilization

### 🟡 C-32 — DNS round robin used as the load balancer
**Signature:** several A records for one service name, with no proxy and no load balancer behind
them. A short TTL treated as a failover mechanism.
**Consequence:** DNS holds no health information and keeps returning addresses for dead servers.
A long-lived Java caller caches the first address forever (Sec. 13.3, p. 233). A reply must fit
in 512 bytes (SRE Ch. 19, p. 274).
**Fix:** put a load balancer that health-checks the pool behind the DNS layer. Squid and Apache
as reverse proxies do not track origin health (Sec. 13.3, p. 235–236).

### 🟡 C-33 — Round robin with a wide range of query costs
**Signature:** a round-robin policy, and a service interface with an unbounded request such as
"return every record for this user".
**Consequence:** simple round robin produces up to a 2-fold CPU spread across tasks. The most
expensive request can consume 1,000 times the resources of the cheapest (SRE Ch. 20, p. 290–291).
**Fix:** cap the work per request with a pagination interface (SRE Ch. 20, p. 291). Then use
weighted round robin, where the backend reports utilization in every response (Ch. 20, p. 296).

### 🟡 C-34 — A least-loaded policy with no error accounting
**Signature:** a client policy that selects the backend with the fewest active requests, and no
term for recent errors.
**Consequence:** a failing task answers faster than a healthy one, because an error is cheaper
than real work, so the policy sends it more traffic. SRE names this sinkholing (Ch. 20, p. 295).
**Fix:** count recent errors as if they were active requests (SRE Ch. 20, p. 295).

### 🟡 C-35 — Every client connects to every backend
**Signature:** a client library with no subset configuration. A health-check interval applied
across the full backend list. A random subset selection.
**Consequence:** memory and CPU pay for connections and health checks and return little. Random
subsetting spreads load between 50 percent and 150 percent of the average at a 10 percent
subset size (SRE Ch. 20, p. 284, p. 286).
**Fix:** use deterministic subsetting with a subset size of 20 to 100 backend tasks. Shuffle the
list, and use a different seed per round (SRE Ch. 20, p. 287–289).

### 🟡 C-36 — Capacity modeled in queries per second
**Signature:** a capacity plan or a quota expressed only in requests per second. A quota
expressed in a static request feature such as a key count. **Modern:** an autoscaling rule
with the same shape.
**Consequence:** the ratio between the proxy metric and the real cost moves when a new version
makes some requests cheaper. A moving target is a poor metric (SRE Ch. 21, p. 297–298).
**Fix:** measure capacity in available resources, and define a normalized cost of a request
(SRE Ch. 21, p. 298). Saturation is one of the four golden signals (SRE Ch. 6, p. 88).

### 🟡 C-46 — The active-request limit does not fit the request duration
**Signature:** a client flow control limit of 100 active requests per backend, against a service
with long-lived requests. A backend with no lame duck state.
**Consequence:** the client treats a slow backend as unhealthy and stops sending. Every backend
task can become unreachable, and the limit cannot separate an unhealthy task from a slow one
(SRE Ch. 20, p. 281–282).
**Fix:** SRE gives 100 active requests as a reasonable limit for most backends (Ch. 20, p. 281).
Add lame duck state, and drain for 10 to 150 seconds before the task exits (Ch. 20, p. 283).

---

## I. The capacity plan

### 🔵 C-37 — The plan cannot be recomputed
**Signature:** a spreadsheet of per-cluster allocations, with no stored inputs and no script. A
plan whose author is the only person who can change it.
**Consequence:** any small change disrupts it. A slipped date or a product decision forces a
cross-check of the whole plan. A slip in one cluster propagates into later quarters (SRE Ch. 18,
p. 251).
**Fix:** encode the inputs and regenerate the plan. SRE names performance data, the demand
forecast, resource supply, and resource pricing (Ch. 18, p. 257–258).

### 🔵 C-38 — The request states the allocation, not the intent
**Signature:** a resource request of the form "X cores in cluster Y", with no statement of the
demand it serves or the redundancy it needs.
**Consequence:** the reasons and the degrees of freedom are lost before the request reaches a
human. The result is manual bin packing, which is NP-hard and has no known bound on the optimal
solution (SRE Ch. 18, p. 252).
**Fix:** state the requirement, not the implementation. Move to level 3 of the intent ladder.
Meet the demand in each region with a stated redundancy (SRE Ch. 18, p. 253–254).

### 🔵 C-39 — No prioritization for the shortfall
**Signature:** a capacity plan with no statement of what the team sacrifices when the resources
do not arrive.
**Consequence:** somebody makes the decision later, under pressure, and makes it inconsistently.
Intent-based planning forces these decisions to be made transparently and consistently (SRE
Ch. 18, p. 255–256).
**Fix:** record which requirements are sacrificed first, and in which order. The output must
list the requirements that nobody could satisfy (SRE Ch. 18, p. 258).

### 🟡 C-40 — A temporary mitigation with no expiry
**Signature:** a throttle, an IP block, a disabled feature, or a static replacement page,
added during an incident. Extra hardware added the same way. No removal date and no owner.
**Consequence:** nothing is as permanent as a temporary fix, and most of Nygard's mitigations
stayed for the next year or two. The cost was lost orders, broken checkout, and doubled hardware
(Sec. 7.6, p. 160).
**Fix:** give every mitigation an expiry date, an owner, and the measurement that permits its
removal. `reverse-branching` owns the removal procedure.

### 🔵 C-47 — The capacity process grows with the service
**Signature:** a manual step for each new cluster, each new customer, or each new quarter. One
spreadsheet that one person maintains by hand.
**Consequence:** the operational work grows with the service. SRE names this work toil, which is
manual, repetitive, and scales linearly with service growth (Ch. 5, p. 75–76).
**Fix:** encode the inputs and generate the plan (SRE Ch. 18, p. 253). Hold toil below 50 percent
of the time of each engineer (SRE Ch. 5, p. 77).

---

## Index by symptom

| Symptom | Check these entries |
|---|---|
| Throughput stops rising while CPU stays low | C-01, C-02, C-10, C-11 |
| The site passed the load test and failed at the launch | C-05, C-06, C-07, C-08, C-17, C-42 |
| Memory grows through the day | C-15, C-16, C-28, C-31 |
| The database server carries thousands of connections | C-10, C-14 |
| A small share of requests fails with no clear cause | C-43 |
| Response time is good on average and bad for some users | C-04, C-23, C-33 |
| The failure appears only after a failover | C-14, C-32 |
| One server is much hotter than the others | C-33, C-34, C-35 |
| Bandwidth or egress cost rose without a traffic rise | C-18, C-20 |
| A query became slow after a year in production | C-24, C-25, C-27 |
| The cache made the system slower | C-28, C-29, C-30 |
| The first requests after a deploy are very slow | C-45, C-46 |
| Work doubles while the site is slow | C-22, C-44 |
| The capacity plan is out of date every quarter | C-37, C-38, C-39, C-47 |
| A mitigation from last year is still enabled | C-40 |
