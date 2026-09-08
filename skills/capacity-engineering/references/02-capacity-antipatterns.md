# The Capacity Antipatterns

**Sources:** *Release It! Design and Deploy Production-Ready Software* — Michael T. Nygard
(Pragmatic Bookshelf, 2007). Ch. 9, "Capacity Antipatterns", Sec. 9.1 to 9.11, p. 175–203.
Supporting pages: Sec. 7.3, p. 152–155, Sec. 7.6, p. 158–160, Sec. 8.6, p. 174.

Each antipattern makes the application do more work than the function needs (Ch. 9, p. 175).
No team selects a design to harm capacity. A team selects a functional design and ignores its
effect on capacity (Sec. 9.11, p. 203).

## How to read this file

Every entry has five fields. **Definition** is what the design is. **Mechanism** is why it
consumes the resource. **Measurable symptom** is the number that tells you to act, and it is
the signal. **Fix** is the remedy that the book states. **Modern** generalizes the 2007
example. Neither book states the **Modern** text.

**A per-unit cost is not the cost. The multiplier is the cost.** Nygard states this as the
first item of the capacity checklist (Sec. 8.6, p. 174). Multiply every per-request byte,
every per-request query, and every per-instance connection by the production volume.

**A symptom without a number is not a symptom.** Report the byte count, the request rate, the
connection total, or the query plan. `01-defining-capacity.md` turns a number into a claim.

## Index of the ten antipatterns

| Antipattern | Section and page | Resource it consumes | Catalog codes |
|---|---|---|---|
| Resource Pool Contention | Sec. 9.1, p. 176–179 | Request threads, database connections | C-10, C-11, C-12, C-13, C-14 |
| Excessive JSP Fragments | Sec. 9.2, p. 180–181 | Class memory, collector time | C-31 |
| AJAX Overkill | Sec. 9.3, p. 182–184 | Request rate, sessions | C-17, C-22 |
| Overstaying Sessions | Sec. 9.4, p. 185–186 | Memory, replication bandwidth | C-15, C-16 |
| Wasted Space in HTML | Sec. 9.5, p. 187–190 | Bandwidth, web server memory | C-20 |
| The Reload Button | Sec. 9.6, p. 191–192 | Request threads, database locks | C-11, C-22 |
| Handcrafted SQL | Sec. 9.7, p. 193–195 | Database I/O, database CPU | C-23, C-25 |
| Database Eutrophication | Sec. 9.8, p. 196–198 | Disk I/O, storage | C-24, C-26, C-27 |
| Integration Point Latency | Sec. 9.9, p. 199–200 | Request threads, response time | C-21 |
| Cookie Monsters | Sec. 9.10, p. 201–203 | Upstream bandwidth, parse CPU | C-18 |

`capacity-defect-catalog.md` holds the searchable signature for each `C-` code.

## 1. Resource Pool Contention (Sec. 9.1, p. 176–179)

**Definition.** More request threads need a pooled resource than the pool holds. Nygard names
the database connection pool as the common case (p. 176).

**Mechanism.** A new database connection takes up to 250 milliseconds, so the pool exists
(p. 176). Contention stays at zero until the thread count passes the resource count. Past
that point the throughput curve flattens at the knee (p. 176). Most pools then block the
caller without a limit. A vicious cycle follows. Contention makes transactions longer, and
longer transactions cause more contention (p. 179). With more than one pool, throughput can
drop exponentially as response time rises (p. 179).

**Measurable symptom.** With 4 database connections and 30 active requests, more than
80 percent of CPU time is spent waiting for a connection (p. 176). Nygard requires three
runtime metrics. Poll them on a schedule and analyze them offline (p. 179).

| Metric | What it proves |
|---|---|
| How often callers block | Contention exists at the current load |
| High-water mark of resources taken since start | The pool size is wrong |
| Resources created and destroyed | The pool churns instead of reusing |

**Fix.** Allow no contention at regular peak, which is a typical day outside the company peak
season (p. 179). Make the pool size equal to the number of request threads (p. 176).
Configure a bounded wait. Nygard names `maxWait` in Jakarta Commons `BasicDataSource` and
`<blocking-timeout-millis>` in JBoss (p. 178). The pool then returns null or throws, and the
application code must handle that result (p. 178). Blocking without a limit guarantees a
stability problem (p. 178). Nygard points from there to Blocked Threads (Antipattern 4.5,
p. 81).

**Compute the fleet total before you enlarge a pool.** 20 machines with 5 instances at
50 connections is 5,000 database connections, and at 1 MB each that is 5 GB of database
server memory (p. 177). During a failover, one node of a database cluster must serve every
query and every connection (p. 179). Size for the degraded topology.

**Modern:** the same arithmetic applies to an HTTP client pool. It applies to a container
fleet behind a connection proxy. It applies to a function that opens one connection per
invocation.

## 2. Excessive JSP Fragments (Sec. 9.2, p. 180–181)

**Definition.** The application stores content as code, so the runtime loads an unbounded
number of generated classes.

**Mechanism.** Each JSP becomes a class in the permanent generation. With no limit on the
number of JSPs, there is no upper bound on the permanent generation size that you need
(p. 180). The flag `-noclassgc` stops the collector from unloading classes. The flag
`-XX:MaxPermSize` caps the region, so the collector works harder to fit classes into it.

**Measurable symptom.** Count the template fragments. One site held more than 25,000 JSP
fragments (p. 181). Watch the permanent generation size through the day. The outcome is
deteriorating performance that becomes indistinguishable from a crash (p. 180).

**Fix.** Remove `-noclassgc`. That trades degeneration for consistently lower performance
(p. 180). Then remove the cause. Do not use code for content. Store the fragments in a
content repository with a cache (p. 181).

**War story — 25,000 fragments (p. 181).** A site treated JSPs as content. Each promotion and
each category description became its own fragment, and nobody retired the old ones. Through
the day the JVM loaded more content classes into the permanent generation, and the collector
worked harder for each one. The fragments held static content only.

**Modern:** the same rule applies to metaspace and to a dynamic class generator. It applies
to a plugin loader. It applies to a build that emits one compiled route per content item.

## 3. AJAX Overkill (Sec. 9.3, p. 182–184)

**Definition.** An asynchronous client feature that shortens think time and multiplies the
request count against the same user population.

**Mechanism.** The delay between page clicks is normally five to ten seconds. With
asynchronous requests the delay between HTTP requests is more like one to three seconds
(p. 182). The user count does not change. The request rate rises. A request that carries no
session identifier makes the application server create a new wasted session every time
(p. 184).

**Measurable symptom.** Requests per user per minute. Sessions created against logins. An
autocomplete field that fires a request every quarter second is the 2007 pattern (p. 183).

| Sub-topic | The rule | Source |
|---|---|---|
| Interaction Design | Apply the technique where it smooths one task in the user's mind | p. 182–183 |
| Request Timing | Send a request when the input changes, or 500 ms after typing stops | p. 183–184 |
| Session Thrashing | Carry the session identifier on every asynchronous request | p. 184 |
| Response Formatting | Reply with JSON, not with HTML fragments | p. 183–184 |

**Fix.** Do not use polling requests for a feature such as autocompletion (p. 184). Configure
session affinity so each asynchronous request reaches the server that holds the session, and
avoid unnecessary session failover (p. 183). Raise the maximum connection count in the web
tier, which Nygard illustrates with Apache `MaxClients` (p. 184).

**Modern:** the same arithmetic applies to a status-polling endpoint and to a websocket
heartbeat. It applies to a client that refetches on every render. It applies to an agent that
polls a job API.

## 4. Overstaying Sessions (Sec. 9.4, p. 185–186)

**Definition.** Sessions stay resident in memory long after the user has gone.

**Mechanism.** The server cannot separate a user who will never click again from a user who
has not clicked yet, so it applies a timeout (Sec. 7.3, p. 153–154). The session therefore
lasts longer than the user, and a session count overestimates the user count (p. 154). A
resident session threatens the system in direct proportion to its tenure in memory (p. 185).

**Measurable symptom.** Heap use that rises through the day. A session count much larger than
the login count. A deployment descriptor with no `session-timeout` element. Nygard calls the
common 30-minute default overkill (p. 185).

**Set the session timeout to one standard deviation past the average think time** (p. 185).
Derive the average from live traffic, and exclude revisits that are hours apart. The measured
targets are about 10 minutes for retail, 5 for a media gateway, and up to 20 for travel
(p. 185).

**Fix.** Keep keys, not whole objects. If you keep whole objects in the session, hold them
through soft references (p. 186). Treat the session as an in-memory cache of persistent
state, so the server can discard it and rebuild it at any time (p. 185–186). One exception
applies. Do not persist financially sensitive data (p. 186).

**War story — the disabled session failover (Sec. 7.6, p. 159–160).** The failover mechanism
serialized each session to a backup server after every page request, and it scales only while
sessions stay small. These sessions held entire shopping carts and search result sets of up
to 2,000 results, so the team had to disable session failover. A user in checkout on a lost
instance then returned to the cart page instead of an order confirmation, and most of those
customers left. Acceptable content is a user ID, a cart ID, and a search key (p. 160).

**Modern:** the same rule applies to a Redis session store and to a server-side token cache.
It applies to a per-user object that a stateful WebSocket server holds.

## 5. Wasted Space in HTML (Sec. 9.5, p. 187–190)

**Definition.** Bytes in the response that carry no information for the user.

**Mechanism.** Each excess byte costs application server CPU to generate. It then costs
bandwidth at every hop, and it costs web server memory while the response stays buffered. A
page of 200 KB instead of 150 KB needs 33 percent more web server RAM (p. 187). Users then
contend for web server connections, and a user who gets none sees a site that is down.

**Measurable symptom.** The byte size of each response, multiplied by the request rate. The
share of that size that is whitespace. Nygard reports pages with 400 KB of tabs, spaces, and
newlines (p. 190).

| Named source | The arithmetic | Fix |
|---|---|---|
| Whitespace | 600 KB of HTML, one third of it newlines and spaces (p. 188) | Filter the whitespace |
| Spacer images | A spacer tag is 53 bytes and `&nbsp;` is 5 bytes, at 12 to 40 places per page (p. 189) | Use `&nbsp;` or CSS |
| Excess HTML tables | A table travels on every page. A style sheet downloads once. The CSS version measured under half (p. 190) | Use CSS layout |

**Fix.** Measure the byte size of every response in the load test. Remove the waste at its
source. A filter is correct when its CPU cost is below the cost of not filtering (p. 188).

**War story — whitespace worth $15,000 a year (p. 188).** A front page was assembled from more
than 100 JSP fragments. Its generated HTML passed 600 KB, and one third of that was newline
characters on lines full of spaces. Each request therefore carried an extra 200 KB of
buffered whitespace on memory-bound web servers. The bandwidth alone would have cost more
than $15,000 a year, before the cost of the extra web servers.

**Modern:** the same rule applies to an uncompressed JSON response and to unread fields. It
applies to a base64 image inside a payload. It applies to a query response with no depth
limit.

## 6. The Reload Button (Sec. 9.6, p. 191–192)

**Definition.** The user retries a slow page, and the server performs the work two times.

**Mechanism.** A full garbage collection can produce a 15-second response time (p. 191). If
the user does not see a page within ten seconds, the user is likely to press Reload (p. 191).
The browser then abandons the previous connection, opens a new socket, and sends a new HTTP
request. Nobody tells the application server to stop the previous request (p. 191). The
connector between the web server and the application server can buffer responses. The server
then may not learn of the dropped connection until it has built and buffered the whole page
(p. 191).

**Measurable symptom.** A request rate that rises while the user count stays flat. Duplicate
transactions from one user identity. Lock waits between two requests that carry the same
session. A transactional second request can block on the first, and can deadlock with it
(p. 191).

**Fix.** Serve pages fast enough that no user presses Reload (p. 191). Then make the duplicate
safe. Prepare the application code for one user to run the same transaction several times
without a deadlock (p. 192).

**Do not serialize requests on source address.** Nygard names this remedy as its own
antipattern and gives two reasons why it fails (p. 192). A caching network or a corporate
gateway proxy makes every request arrive from one source address. And the user is no longer
waiting for the first response, so serializing makes the user wait longer and press Reload
again.

**Modern:** the same mechanism appears in a mobile client that retries on a spinner. It
appears in a double form submit, and in a job runner that resends. An idempotency key is the modern form
of the rule at p. 192. `data-systems-design` owns it, and `self-healing-apis` owns retries.

## 7. Handcrafted SQL (Sec. 9.7, p. 193–195)

**Definition.** Developer-written one-off SQL that leaves the object-relational mapping.

**Mechanism.** An ORM generates predictable, repetitive SQL, and a competent DBA can tune the
database for it (p. 193). Handcrafted SQL is idiomatic and unpredictable. Tuning cannot serve
it, and tuning for it does no good to the rest of the application and might harm it (p. 194).

| Named defect | What it does | Page |
|---|---|---|
| Join on nonindexed columns | Forces a table scan, the slowest way to find a row in a large table | p. 193–194 |
| Join too many tables | Multiplies the work of the query plan | p. 193 |
| Use a set language one row at a time | Joins five tables to select one row, then issues that query 100 more times | p. 193 |
| Exercise exotic features | An eight-way union whose plan held about forty table scans | p. 193–194 |

**Measurable symptom.** Read the query plan and count the table scans. Compare the row count
of the test database against production. A gain measured on a development database can
disappear against a different query plan in production (p. 195).

**Fix.** Minimize handcrafted SQL. Try an index, a hint, or a view inside the database first.
Apply the laugh test. A query that does not pass it does not go into production (p. 195).
Verify every gain against production-sized data, scrubbed by replacing private data with a
random scramble of characters (p. 194). Where no realistic data exists, spend an hour or two
on a data generator (p. 194). Query-by-example belongs in the reporting system (p. 194).

**War story — the eight-way union (p. 194).** One handwritten query took a full page to print.
Every subselect in the union joined five tables on nonindexed columns, and the query plan
held about forty table scans. The opposite case stands beside it. DBAs report processes cut
from eighteen hours to three minutes by one index or by an analysis of table statistics. An
order-of-magnitude gain is common, and it lives in the database.

**Modern:** the same rule applies to a raw SQL string beside an ORM call. It applies to a
filter API that builds predicates from client input. It applies to a query that a code
generator or a language model produced. A generated query still needs the laugh test.

## 8. Database Eutrophication (Sec. 9.8, p. 196–198)

**Definition.** The database ages until it can no longer serve the application. Nygard names
the antipattern after a lake that fills with sludge until nothing can live in it (p. 196).

**Mechanism.** Three separate causes reach the same result.

| Named remedy | The cause it removes | Page |
|---|---|---|
| Indexing | An ORM mapping file triggers no database review, so an unindexed association reaches production | p. 196 |
| Partitioning | A wrong growth guess spreads a table across physical extents until disk I/O dominates response time | p. 197 |
| Historical data | Reports and old rows compete with transactions for the constrained resource | p. 197–198 |

**Measurable symptom.** State the row size, the rows per day, and the retention period for
every table. A scan that is invisible on development data becomes a wait of minutes after a
year or two (p. 196). On a small table a scan can even be the fastest plan, which hides it.

**Fix.** Index every column that is the target of an association in the ORM mapping (p. 196).
The first iteration of indexes is the developer's responsibility, not only the database
architect's (p. 198). Partition on a column with a small finite set of values, where each
value marks a cluster of related rows (p. 197). One partition can then move to another
physical extent while other partitions stay in heavy use, without downtime (p. 197). Run a
rigorous regimen of data purging (p. 198).

**Do not mix transactions and reporting** (p. 198). An OLTP schema is optimized for fast
inserts of transactional records, and it is bad for reports and ad hoc queries. Serve reports
from a star schema. Restrict a user-facing report to the last ninety days or six months
(p. 198).

**War story — 1 GB a year becomes 1 GB a day (p. 197).** One release of an order management
system changed the audit log volume by a factor of one thousand. The storage layout no longer
held, and the tables spread across physical extents until disk I/O dominated the response
time. A single release can invalidate a storage assumption, and partitioning is the defense
that allows a repair without downtime.

**Modern:** the same rules apply to a partitioned Postgres table. They apply to a retention
policy in a cloud data warehouse. They apply to change data capture that moves reporting off
the write path.

## 9. Integration Point Latency (Sec. 9.9, p. 199–200)

**Definition.** The latency of one remote call, multiplied by the number of calls behind one
user action.

**Mechanism.** A remote call takes at least 1,000 times as long as a local call (p. 199). The
caller therefore takes at least as long to respond as the remote system (p. 199). Location
transparency is discredited for two reasons. Remote calls have different failure modes, which
include network failure, failure in the remote process, and version mismatch. And it leads to
chatty interfaces, where one interaction costs many method calls (p. 199). A thread that waits
still holds memory, CPU time slices, database connections, and row or page locks (p. 199).
Performance problems for individual users become capacity problems for the whole system
(p. 199).

**Measurable symptom.** Count the calls behind one user action. The 1+N pattern is one call to
find the membership of a collection, plus one or more calls per member (p. 193 footnote,
p. 200). Measure the same action from the farthest client location.

**Fix.** Expose yourself to latency as seldom as possible (p. 199). Add a coarse method that
returns a collection of Summary Objects, each carrying exactly the fields that the caller
needs (p. 200). Avoid chatty remote protocols, and protect request-handling threads
(Sec. 8.6, p. 174). `self-healing-apis` owns the Circuit Breaker and the timeout policy.

**War story — RMI Across the Seas (p. 200).** A fat-client Java system used RMI against an
enterprise data warehouse. Expanding one node of a tree control took under a second in the
United States and twenty minutes from the United Kingdom. The client made one call for the
child collection and then three calls per child, which is the 1+N pattern. UK users were
asking for a local warehouse, a multimillion-dollar investment. One new method returning
Summary Objects with three fields gave those users subsecond response.

**Modern:** the same arithmetic applies to a REST endpoint called inside a loop. It applies to
a resolver that fans out per item. It applies to an ORM lazy load across a service boundary,
and to an agent that calls one tool per row.

## 10. Cookie Monsters (Sec. 9.10, p. 201–203)

**Definition.** The application stores whole objects in HTTP cookies instead of identifiers.

**Mechanism.** RFC 2109 defined cookies for session management. Cookies were meant to send
small chunks of data, less than 100 bytes or so, which is all a session identifier needs
(p. 202). A 4 KB serialized object crosses the wire two times, and sometimes four times, and
the server parses it two times per request (p. 202). It travels on the user's upstream
bandwidth, which is much smaller than the downstream bandwidth (p. 202). The serialized form
also outlives the code that wrote it, so a code change invalidates held cookies (p. 201–202).

**Measurable symptom.** Measure the cookie and header bytes on every request. Count the
deserialization errors. Nygard reports that these errors become a routine part of doing
business (p. 202).

**Fix.** Use cookies for identifiers, not entire objects. Keep session data on the server,
where a malicious client cannot alter it (p. 203). Assume three client behaviors. The client
can lie, can send stale or broken cookies, and can send no cookies at all (p. 203).
`security-engineering` owns the trust rules for client-supplied data.

**War story — the serialized shopping cart (p. 201–203).** A team added an interceptor that
serialized an anonymous user's shopping cart into a cookie, to avoid database rows for
visitors who might never return. The serialized cart could then be months or years old, so a
code change made it invalid before the user returned. Products in it went missing, so the
interceptor also had to handle referential integrity and code versions. A malicious user
could edit the cookie and set every price to one cent. All of that replaced one purge job.

**Modern:** the same arithmetic applies to a large signed token in an `Authorization` header.
It applies to a token that carries a permission list. It applies to client state in local
storage that the client sends on every call.

## How to run this file

Run it against a design, a difference, or a configuration file. Report only what applies.

1. Name the resource that the change consumes. Read `01-defining-capacity.md` first.
2. Find the multiplier. Multiply the per-unit cost by the production volume.
3. Match the design against the ten entries above. Use the index table.
4. Report the measurable symptom for each match, with its number.
5. Give the fix that the book states, and cite the section and the page.

**"Not applicable" is a valid result.** A change that adds no request, no query, no byte, and
no session triggers no entry here. An invented finding is worse than no finding.

`03-capacity-patterns.md` gives the four remedies that elevate a constraint.
`04-load-testing.md` gives the test that produces the numbers this file asks for.
