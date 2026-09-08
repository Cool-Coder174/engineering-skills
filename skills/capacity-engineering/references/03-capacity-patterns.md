# The Capacity Patterns

**Sources:** *Release It! Design and Deploy Production-Ready Software* — Michael T. Nygard
(Pragmatic Bookshelf, 2007). Ch. 10, Capacity Patterns, Sec. 10.1 to Sec. 10.5, p. 204–217.
Supporting citations from the same book: Sec. 7.6, p. 158–160, and Sec. 8.2, p. 162–164.
Also Sec. 8.5, p. 166–173, Sec. 8.6, p. 174, and Sec. 9.1, p. 176–179.

Nygard gives four capacity patterns. This file states the signal that selects each pattern,
the price that each pattern charges, and the measurement that confirms it.

---

## 1. Read this before you select a pattern

**A pattern applied to a nonconstraint gives no capacity.** Only the constraint sets the
capacity of a system (Sec. 8.2, p. 163). Name the constraint first. `01-defining-capacity.md`
gives the method that names it.

**Optimization arrives late. A design choice arrives early.** Nygard cites Hoare on premature
optimization and then states the limit of optimization (Ch. 10, p. 204). Optimization raises
the speed of one routine by a percentage, and it does not produce a better design. You would
never optimize your way from a bubble-sort to a quicksort (Ch. 10, p. 204).

**The multiplier selects where a change pays.** Examine the multiplier effects to find the
right point for an improvement. The system can pay a cost a million times a day for a benefit
that arrives once a week. That cost sits on the wrong side of the multiplier (Sec. 10.5,
p. 217).

**War story — the ten-million-dollar holiday budget (Ch. 10, p. 204).** Nygard saw code that
performed poorly drive an organization to budget ten million dollars for extra hardware. The
hardware was to survive one holiday season, which was about one week of sales. The team fixed
a few of the capacity antipatterns and applied two of these patterns, Precompute Content and
Use Caching Carefully. The organization then avoided the cost. The story proves that the
capacity came from the design, and not from late optimization.

### Table 1 — Pattern selection

| Pattern | Signal that selects it | Price | Measurement that confirms it |
|---|---|---|---|
| Pool Connections (Sec. 10.1, p. 206) | Connection setup appears in the transaction time. Setup takes 400 to 500 milliseconds | An undersized pool creates contention. An oversized pool stresses the database server | Wait time for a connection, and the high-water mark |
| Use Caching Carefully (Sec. 10.2, p. 208) | The system reads the same object often, and the source data changes rarely | Memory that requests need. A stale read. A flush storm | Hit rate for each cached class, and the maximum memory in use |
| Precompute Content (Sec. 10.3, p. 210) | The content changes much less often than the pages are generated | Storage, a publish step, and a slow first request after a restart | The ratio of renders per day to content changes per day |
| Tune the Garbage Collector (Sec. 10.4, p. 214) | The collector uses about 10 percent of the runtime at production volume | The tuning belongs to one release. You repeat it | Collector time as a percentage of runtime |

---

## 2. Pool Connections (Sec. 10.1, p. 206–207)

**Signal.** Connection setup appears in the transaction time. A new database connection needs
a TCP connection, database authentication, and database session setup. Together these can
easily take 400 to 500 milliseconds (Sec. 10.1, p. 206). Only the start of a new thread costs
more than the creation of a database connection (Sec. 10.1, p. 206). A connection pool
removes up to 500 milliseconds from every transaction (Sec. 10.5, p. 217).

**Nygard states that there is no excuse to omit connection pooling, except a poor
implementation** (Sec. 10.1, p. 206). Nearly every language permits it (Sec. 10.1, p. 206).

### The price

**Price 1 — one bad connection causes a disproportionate share of the errors.** A connection
can enter a bad state. That connection returns to the pool fast, because it fails fast. The
good connections stay out longer, because they do real work. So the bad connection is more
often available. Nygard states the result in one sentence: "One bad connection out of ten will
cause more than 10% of requests to error out" (Sec. 10.1, p. 206).

**Price 2 — the pool size is a two-sided error.** An undersized connection pool leads to
resource pool contention (Sec. 10.1, p. 206, and Antipattern 9.1, p. 176). An oversized
connection pool puts excess stress on the database servers (Sec. 10.1, p. 206).

**Price 3 — the fleet total is much larger than the pool.** Compute the total before you
enlarge a pool. 20 machines, with 5 instances each, at 50 connections each, is 5,000 database
connections. That is 5 GB of database server memory for the connections alone (Sec. 9.1,
p. 177). Under a failover, one node serves every query and every connection (Sec. 9.1, p. 179).

### Table 2 — Connection checkout strategy

| Strategy | How it works | What it gives | What it costs |
|---|---|---|---|
| Per-page (p. 206–207) | One connection serves the whole page. The code returns it when the page completes | Safer against deadlock, because one checkout order applies to every request | A higher ratio of connections to request threads, because each connection stays out longer |
| Per-fragment (p. 207) | Each fragment takes its own connection, does the work, and returns the connection | Higher throughput. Fewer connections per request thread. No fragment needs global transaction knowledge | More susceptible to deadlock |
| Hybrid (p. 207) | Each fragment manages its own connections. One database transaction wraps the whole page | The deadlock safety of per-page, with the isolation of per-fragment | The larger pools of per-page. A fragment can see uncommitted data from an earlier fragment, which is hard to diagnose |

A transaction manager usually implements the per-page model, with one transaction for the
whole page. A page built from many fragments may not permit that model at all (Sec. 10.1,
p. 207).

### The rules (Remember This, p. 207)

1. Pool the connections. Connection pooling is basic.
2. Give every checkout call a timeout. Do not allow a caller to block forever.
3. Define what the caller does when it gets no connection back.
4. Size the pool for maximum throughput.
5. Monitor the calls to the pool, and record how long the threads wait.

**You must monitor the pools for contention, or this capacity enhancer becomes a killer**
(Sec. 10.1, p. 207).

**Measurement.** Nygard requires three runtime metrics for a pool. They are how often callers
block, the high-water mark of resources taken since start, and the count of resources created
and destroyed (Sec. 9.1, p. 179). The target at a regular peak is zero contention, where a regular
peak is a typical day outside the peak season (Sec. 9.1, p. 179).

**Modern.** The same arithmetic applies to an HTTP client pool, a gRPC channel pool, and a
connection proxy. A function that opens one connection for each invocation repeats the early
Perl CGI pattern that Nygard describes (Sec. 10.1, p. 206). Neither book states this.

**Defect codes.** C-10 to C-14 in `capacity-defect-catalog.md`.

---

## 3. Use Caching Carefully (Sec. 10.2, p. 208–209)

**Caching carefully means three commitments. A bounded size. A stated invalidation rule. A
stated hit-rate expectation.** A cache without all three is not this pattern. Nygard opens the
section with the warning that a misused cache creates new problems (Sec. 10.2, p. 208).

**Signal.** The system reads the same object often, and the source data changes rarely.
Nygard states the bet that a cache makes. One generation of the item, plus the hash and the
lookup, costs less than a generation on every request (Sec. 10.2, p. 208).

### The price

**Price 1 — a cache with no limit causes the slowdown that it was meant to prevent.** A cache
that does not limit its memory consumption eats the memory that the system needs. The garbage
collector then spends more and more time to recover enough memory to process requests. The
cache itself causes the serious slowdown (Sec. 10.2, p. 208).

**Price 2 — a low hit rate buys nothing.** A cache with a very low hit rate gives no
performance gain, and it can be slower than no cache (Sec. 10.2, p. 208).

**Price 3 — every cache risks stale data.** Nygard states the rule as one sentence: "Every
cache should have an invalidation strategy to remove items from cache when their source data
changes" (Sec. 10.2, p. 209).

**Price 4 — a flush is expensive.** Frequent flushes produce attacks of self-denial
(Sec. 10.2, p. 209).

### Table 3 — The five decisions

| Decision | The rule | Source |
|---|---|---|
| Maximum size | The maximum memory of all application-level caches is configurable | Sec. 10.2, p. 208 |
| Hit rate | Monitor the hit rate for the cached items. A low rate can be slower than no cache | Sec. 10.2, p. 208 |
| What not to cache | Do not cache a trivial object. Do not cache an object used once in the life of a server | Sec. 10.2, p. 208–209 |
| Invalidation | Every cache removes an item when its source data changes | Sec. 10.2, p. 209 |
| Flush rate | Limit how often a flush can start | Sec. 10.2, p. 209 |

### State the hit-rate expectation before you merge the cache

**Write the expected hit rate as a number, and then measure the real rate.** Nygard gives no
threshold number for a hit rate, so the number is a local decision. He names two cases that
disqualify a cache outright. An object used only once during the life of a server gains
nothing from a cache. An object that is cheap to generate gains nothing either (Sec. 10.2,
p. 208).

Nygard reports content caches that held hundreds of entries, where each entry was a single
space character (Sec. 10.2, p. 208). One JSP fragment cached the outcome of a Boolean test on
a user profile, and the cached object served one user (Sec. 10.2, p. 208–209). The bookkeeping
cost and the loss of free memory outweigh the gain for a trivial object (Remember This,
p. 209).

### Table 4 — Invalidation method by fleet size

| Fleet size | Method | Source |
|---|---|---|
| Ten or twelve application servers | Point-to-point notification works well | Sec. 10.2, p. 209 |
| Hundreds of application servers | Point-to-point unicast is not effective. Use a message queue or multicast notification | Sec. 10.2, p. 209 |

With multicast, make sure that the application servers do not all hammer the database at the
same time to reload the invalidated item (Sec. 10.2, p. 209).

### When a second cache level earns its cost

Multilevel caching keeps the most frequently accessed data in memory and uses disk storage
for a secondary cache. Nygard names three cases where it works well (Sec. 10.2, p. 208–209).

1. The objects to cache are extremely large, such as images.
2. The working set is larger than the memory that you can hold.
3. A fetch of uncached data crosses a WAN connection.

**Java mechanism.** Build caches with `SoftReference` objects to hold the cached item. The
garbage collector may then reap an object that is reachable only through a soft reference. The
cache helps the collector reclaim memory instead of preventing it (Sec. 10.2, p. 208, and
Sec. 10.5, p. 217).

**Precomputed results reduce or eliminate the need for caching** (Sec. 10.2, p. 209). Read
Section 4 of this file before you add a cache to a render path.

**Modern.** The same five decisions apply to an external cache such as Redis or Memcached, to
an HTTP cache header, and to a CDN edge cache. The invalidation correctness rules belong to
`data-systems-design`. Neither book states this.

**Defect codes.** C-28, C-29 and C-30 in `capacity-defect-catalog.md`.

---

## 4. Precompute Content (Sec. 10.3, p. 210–213)

**Signal.** The content changes much less often than the pages are generated. Nygard adds a
second condition that makes the pattern strong. You can identify the precise point in time
when the content changes (Sec. 10.3, p. 210).

**The worked ratio.** A retail category menu appears on almost every page. The code that
queries the categories and renders the fragment probably executes a million times a day. The
top-level categorization changes once every three months. Small changes happen once a week
(Sec. 10.3, p. 210).

**A cache on the query does not remove the render cost.** You can cache the results of the
database query. The render of the HTML still takes a long time, because of the sheer number
of strings (Sec. 10.3, p. 210).

**The rule (Remember This, p. 213).** Factor the cost to generate the content out of the
individual requests and into the deployment process.

### The price

1. Precomputed content needs storage space for each piece of computed content (p. 212).
2. There is a runtime cost to map an identifier to a file and to read the file. For commonly
   used content, that cost can motivate a memory cache for the content itself (p. 212).
3. The generation cost still exists. It moves to the moment when the content changes (p. 212).

**Personalization works against precomputed content.** If entire pages are personalized, then
precomputed content is impossible. If only a few fragments are personalized, then you
precompute the majority of the page and punch out a hole for the personalized content
(Sec. 10.3, p. 212). Nygard gives the split for a retail home page. About 100 bytes are
customer specific. The remaining 100 KB are exactly the same for every customer (Sec. 10.3,
p. 213).

**Precomputed content is not an all-or-nothing choice.** Precompute the high-traffic areas,
and leave the less visited pages fully dynamic (Sec. 10.3, p. 213).

### Table 5 — Precomputed content against an in-memory fragment cache

Nygard states that precomputing content is different from caching (Sec. 10.3, p. 212).

| Question | Precompute content | In-memory fragment cache |
|---|---|---|
| What does it trade? | It moves the generation cost out of the request and into the deployment process (p. 213) | It balances application server response time against application server memory (p. 212) |
| Behavior under memory pressure | The content sits in storage, so requests keep their transient memory | The server thrashes the cache and works with reduced transient memory. A Java application server becomes very slow (p. 212) |
| Behavior after a restart | The stored result is already present | The caches are cold. The first requester might wait minutes for a single page (p. 212–213) |

Nygard states the boundary. In-memory caching has its place, and storing rendered page
fragments for large amounts of content is not the right way to use caching (Sec. 10.3, p. 213).

**War story — Slashdot and Fark (p. 210–211).** Both news portal sites precompute their main
pages. The stories change at least hourly, and sometimes more often. The comment totals change
every minute. The main page of each site is still a template that includes a handful of large
pieces of precomputed content. The sites get their liveness because they recompute the content
every few minutes. Each version of the page is still viewed hundreds or thousands of times
before the next update. The story proves that precomputation is compatible with apparent
liveness.

**War story — the Profanity Masker (p. 211–212).** At the retail launch of Chapter 7, the
garbage collection statistics showed almost 10 MB of garbage for each page request. The cause
was a custom ATG droplet named the Profanity Masker. The team applied it around the product
name, the short description, the long description, the specifications, each music track name,
and each movie actor name. A single product detail page could carry twenty instances, at more
than five million page views a day. The droplet tokenized every piece of text, compared each
word against a `Vector` of eleven dirty words, and then stitched the text back together. All
of that content was published nightly, so nothing could change during the day. The postscript
matters as much as the defect. The team asked whether it could stop the masking. The business
sponsor for content objected. The product descriptions, album names, sample lyrics, and song
titles are copyrighted material from a data vendor, and the contract forbids any alteration.
The team removed the Profanity Masker. Nobody ever learned why someone wrote it. The story
proves the multiplier effect. It also proves that a code removal can carry a contractual
constraint that appears only when someone proposes the change.

**Modern.** The same signal selects static site generation, an incremental build, a
materialized view, and a publish-time push to a CDN. The recompute trigger is the moment when
the source data changes. Neither book states this.

**Defect codes.** C-19 in `capacity-defect-catalog.md`. Check C-20 too, because the response size obeys the same multiplier.

---

## 5. Tune the Garbage Collector (Sec. 10.4, p. 214–217)

**Signal.** An untuned application that runs at production volumes and traffic will probably
spend 10 percent of its time collecting garbage. You should reduce this to 2 percent or less.
In Java applications, this tuning is the quickest and easiest way to see a capacity
improvement (Sec. 10.4, p. 214).

**How the collector works (p. 214).** Object life spans have a bimodal distribution. Most
objects are ephemeral, and Sun calls this infant mortality. A much smaller population lives as
long as the program. Objects allocate in the eden space, and a survivor moves into a survivor
space. Both spaces sit in the young generation. Later phases promote survivors into the
tenured generation, which the collector examines much less often. The permanent generation
holds the class and method definitions.

### The procedure (p. 214–215)

1. Pass the `-verbosegc` argument to the JVM at start-up.
2. Direct standard out somewhere, because the collector reports go to the console output. For
   an application server this usually needs a change to a start-up script.
3. On Java 5 or later, use `jconsole` instead. The Memory tab shows the heap usage by
   generation and by space, and the time spent in garbage collection.
4. Observe the collection patterns.
5. Ensure a sufficient heap size.
6. Adjust the ratios that control the relative sizes of the generations.

### The price

**Price 1 — you can only tune the collector in production.** User access patterns make a huge
difference to the optimal settings, so development and QA cannot produce them (Remember This,
p. 217).

**Price 2 — the tuning belongs to one release.** Collector behavior derives entirely from the
behavior and the demand patterns of the application. Each code release changes the environment
in some way. Even a small release can induce new user behavior. Settings that are perfectly
tuned for one release can be totally wrong for the next release (Sec. 10.4, p. 215).

**Price 3 — the work repeats on a schedule.** You need to tune the collector after each major
application release. With an annual demand cycle, you also tune at different times of the
year, as user traffic shifts between features (Remember This, p. 217). Nygard requires a
routine process for this retuning, and he points to Transparency (Ch. 17, p. 265) for it.

**One benefit that is not capacity.** The tuning exercise often brings memory leaks to your
attention (Sec. 10.4, p. 215).

### Do not pool ordinary objects

**The only objects worth pooling are external connections and threads. For everything else,
rely on the garbage collector** (Remember This, p. 217). Nygard names network connections,
database connections, and worker threads as the objects that are really expensive to create
(Joe Asks, p. 216). A system that avoids object creation adds so much complexity and
bookkeeping that it removes any possible performance gain (Joe Asks, p. 216).

Nygard's measurement formatted 50,000 names with a `NameFormatter` on JDK 1.4.2. The pooled
configuration used the Jakarta commons-pool package, and the unpooled configuration created
50,000 individual objects (Joe Asks, p. 216).

| Platform and CPU speed | Overhead, pooled | Overhead, disposable |
|---|---|---|
| Windows XP Pro, 1.86 GHz | 20.30% | 10.17% |
| Linux 2.6.14, 2.66 GHz | 31.46% | 23.42% |
| Mac OS 10.4.4, 1.67 GHz | 24.69% | 15.69% |

The pooled overhead is higher on every platform. The bookkeeping overhead of the pool
overwhelms the expense of the object construction (Joe Asks, p. 216).

**Modern.** Other managed runtimes have their own collector controls and their own memory
regions. The three rules do not change. Measure the collector time as a percentage. Tune at
production traffic. Retune after a major release. Neither book states this.

**Defect codes.** C-31 in `capacity-defect-catalog.md`.

---

## 6. Order of application, and when to stay silent

**Apply the pattern that matches the named constraint. Apply nothing else.** Table 1 gives the
signal. If no signal matches the constraint, this file has no answer for it.

Two ordering rules follow from the chapter.

1. Precompute before you cache a render. Precomputed results reduce or eliminate the need for
   caching (Sec. 10.2, p. 209).
2. Size and monitor the pool before you enlarge it. An unmonitored pool becomes a killer
   (Sec. 10.1, p. 207).

**This file stays silent in three cases.** No measurement names a constraint, and
`01-defining-capacity.md` runs first. The change adds no pool, no cache, no render path, and
no long-lived process. Or the pattern would apply to a resource that is not the constraint
(Sec. 8.2, p. 163). "Not applicable" is a valid result. It is better than a pattern applied on
a guess.

---

## 7. Related files

- `01-defining-capacity.md` — the four numbers, and the method that names the constraint.
- `02-capacity-antipatterns.md` — the defects that these patterns repair, from Ch. 9.
- `04-load-testing.md` — the test that produces the evidence for a pattern.
- `05-intent-based-capacity-planning.md` — the plan that regenerates when an input changes.
- `06-load-balancing-and-utilization.md` — the distribution method and the utilization signal.
- `capacity-defect-catalog.md` — the codes C-10 to C-14, C-19, C-20, and C-28 to C-31.
- `../SKILL.md` — Section 6 carries the pattern table, and Section 7 the safety limits.

Two boundaries. `data-systems-design` owns cache invalidation correctness, and
`self-healing-apis` owns the behavior of the system above the limit.
