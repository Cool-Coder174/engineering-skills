# Back-of-the-Envelope Estimation

**Sources:** *System Design Interview* Ch. 2 (Xu); *Grokking the System Design Interview*,
Step 3; latency numbers originally from Jeff Dean (Google).

Estimation is what turns a diagram into a design. It answers whether you need one machine
or a hundred, one database or a sharded cluster — and it is the only honest way to reject
an over-engineered proposal.

---

## 1. Powers of two

Data volume units, which is where every calculation bottoms out:

| Power | Approximate value | Name | Short |
|---|---|---|---|
| 2^10 | 1 thousand | 1 Kilobyte | 1 KB |
| 2^20 | 1 million | 1 Megabyte | 1 MB |
| 2^30 | 1 billion | 1 Gigabyte | 1 GB |
| 2^40 | 1 trillion | 1 Terabyte | 1 TB |
| 2^50 | 1 quadrillion | 1 Petabyte | 1 PB |

A byte is 8 bits. An ASCII character is 1 byte.

**Useful constants to memorize:**
- 1 day = 86,400 seconds ≈ **10^5 s** (this approximation is accurate enough and makes the
  arithmetic trivial)
- 1 month ≈ 2.5 × 10^6 s
- 1 year ≈ 3.15 × 10^7 s ≈ 365 days
- 1 million requests/day ≈ **12 QPS**
- 100 million requests/day ≈ 1,160 QPS

---

## 2. Latency numbers every programmer should know

Dr. Jeff Dean's figures. Some are dated as hardware improves, but the **relative
magnitudes** are what matter and those have held.

| Operation | Time |
|---|---|
| L1 cache reference | 0.5 ns |
| Branch mispredict | 5 ns |
| L2 cache reference | 7 ns |
| Mutex lock/unlock | 100 ns |
| Main memory reference | 100 ns |
| Compress 1 KB with Zippy | 10,000 ns = 10 µs |
| Send 2 KB over 1 Gbps network | 20,000 ns = 20 µs |
| Read 1 MB sequentially from memory | 250,000 ns = 250 µs |
| Round trip within the same datacenter | 500,000 ns = 500 µs |
| Disk seek | 10,000,000 ns = 10 ms |
| Read 1 MB sequentially from network | 10,000,000 ns = 10 ms |
| Read 1 MB sequentially from disk | 30,000,000 ns = 30 ms |
| Send packet CA → Netherlands → CA | 150,000,000 ns = 150 ms |

*(1 ns = 10⁻⁹ s; 1 µs = 1,000 ns; 1 ms = 1,000 µs = 1,000,000 ns)*

**The conclusions that actually drive design decisions:**
- **Memory is fast; disk is slow.** Roughly a 100× gap, sequential; far worse for seeks.
- **Avoid disk seeks.** A seek costs about as much as reading a megabyte sequentially.
- **Simple compression is fast** — 10 µs per KB.
- **Compress before sending over the network.** Compression is cheap relative to transfer.
- **Cross-region traffic is expensive.** ~150 ms round trip intercontinental means a design
  with several sequential cross-region hops cannot meet a 200 ms p99, no matter how fast
  the code is.
- A same-datacenter round trip (500 µs) is ~5,000× a main memory reference. **Every network
  hop you add to a request path is a decision, not an implementation detail.**

---

## 3. Availability numbers

Availability is measured in "nines". An SLA formally defines the uptime you commit to;
major cloud providers set theirs at 99.9% or above.

| Availability | Downtime/day | Downtime/week | Downtime/month | Downtime/year |
|---|---|---|---|---|
| 99% | 14.40 min | 1.68 h | 7.31 h | 3.65 days |
| 99.9% | 1.44 min | 10.08 min | 43.83 min | 8.77 h |
| 99.99% | 8.64 s | 1.01 min | 4.38 min | 52.60 min |
| 99.999% | 864 ms | 6.05 s | 26.30 s | 5.26 min |
| 99.9999% | 86.4 ms | 604.8 ms | 2.63 s | 31.56 s |

**How to use this in a design:**
- **Serial dependencies multiply.** A request touching four services at 99.9% each has a
  ceiling of 0.999⁴ ≈ **99.6%** — worse than any single component. This is the argument for
  fewer hops, for caching, and for graceful degradation.
- **99.99% leaves ~4.4 minutes per month.** That budget is consumed by deploys, migrations,
  and failovers, not just outages. If deploys cause 30 seconds of unavailability and you
  deploy daily, you have already spent 15 minutes.
- Choose the target from the business need, then check whether the architecture can even
  express it. Three nines with a manual failover procedure is not three nines.

---

## 4. Estimation formulas

### QPS
```
DAU               = MAU × (fraction active daily)
Actions/day       = DAU × actions per user per day
Average QPS       = actions/day ÷ 86,400        (≈ ÷ 10^5)
Peak QPS          = average QPS × peak factor   (2–10×; use 2× as the default,
                                                 higher for event-driven or diurnal loads)
```

### Storage
```
Storage/day       = writes/day × average record size
Storage @ horizon = storage/day × 365 × retention years
With replication  = × replication factor (typically 3)
With overhead     = × 1.2–1.5 (indexes, metadata, fragmentation, free space)
```
Estimate the **dominant** contributor first. If 10% of records carry 1 MB of media and the
rest are 200 bytes of text, the media term is the entire answer — compute it and move on.

### Bandwidth
```
Ingress = write QPS × average payload size
Egress  = read QPS × average response size
```
Egress is usually the larger number and usually the larger bill. For media-heavy systems
it is the primary argument for a CDN.

### Memory / cache
```
Working set  = requests/day × fraction of distinct hot items × item size
Cache size   = working set × 1.2   (overhead)
```
The 80/20 heuristic: roughly 20% of items generate roughly 80% of traffic, so caching the
hot 20% is where most of the benefit is. Verify against real access distributions when you
have them — the true distribution is often far more skewed.

### Server count
```
Servers = peak QPS ÷ QPS per server
```
Typical per-server capacity (validate by measurement; these are order-of-magnitude only):
simple API on modern hardware ~500–1,000 QPS; heavy computation ~50–200 QPS; static content
far higher. Then add headroom for failure (N+1 or N+2) and for the peak *of the peak*.

### Connections
```
Concurrent connections = QPS × average request duration (Little's Law)
```
Little's Law is the most useful formula on this page. 1,000 QPS with 200 ms average
duration means **200 concurrent requests in flight** — which is what sizes your connection
pools and thread pools, and which is why a dependency slowing from 200 ms to 2 s multiplies
your in-flight count by 10 and exhausts the pool.

---

## 5. Worked example (Xu's Twitter estimate)

**Assumptions** *(stated up front so they can be challenged)*
- 300 million monthly active users
- 50% use the product daily
- 2 posts per user per day on average
- 10% of posts contain media
- Media averages 1 MB
- Data retained 5 years

**QPS**
```
DAU        = 300M × 50%                     = 150M
Post QPS   = 150M × 2 ÷ 86,400              ≈ 3,500 QPS
Peak QPS   = 3,500 × 2                      ≈ 7,000 QPS
```

**Storage (media dominates)**
```
Average post: post_id 64 B + text 140 B + media 1 MB
Media/day  = 150M × 2 × 10% × 1 MB          = 30 TB/day
5 years    = 30 TB × 365 × 5                ≈ 55 PB
```

**What this tells you immediately:** 7,000 peak write QPS is comfortably beyond a single
database, so partitioning is required. 55 PB of media does not belong in a database at all
— it belongs in object storage with references in the database. Both conclusions come from
arithmetic, not opinion, and both would have been guesses without it.

---

## 6. A second worked example (read-heavy, showing the read/write asymmetry)

**Assumptions:** URL shortener, 100M new URLs/month, 100:1 read:write, 5-year retention,
500 bytes per record.

```
Write QPS   = 100M ÷ (30 × 86,400)          ≈ 40 QPS
Read QPS    = 40 × 100                      = 4,000 QPS
Records     = 100M × 12 × 5                 = 6 billion
Storage     = 6B × 500 B                    = 3 TB
Cache (20%) = 4,000 × 86,400 × 20% × 500 B  ≈ 35 GB/day of hot data
```

**What this tells you:** writes are trivial (40 QPS runs on one modest server), reads are
100× writes, and the hot working set fits in memory on a single cache node. The design
follows directly — read replicas plus a cache, and no sharding for years. **A design that
proposed sharding here would be over-engineering, and the numbers are how you prove it.**

---

## 7. Rules for doing this well

1. **Round aggressively.** Use 10^5 for a day. Precision is not the point and chasing it
   costs the time you need for actual design.
2. **Write assumptions down.** Every estimate is a function of assumptions; a reviewer
   should attack the assumption, not the arithmetic.
3. **Label units.** "5" is ambiguous. "5 MB" is not. Unit confusion is the most common
   estimation error.
4. **Estimate the dominant term first.** Usually one term is 100× the others; find it and
   the rest is noise.
5. **Sanity-check against reality.** If your estimate says you need 40,000 servers, either
   the estimate or the design is wrong.
6. **Prefer measurement.** In real work, production metrics beat estimation. Estimate for
   what does not exist yet; measure everything that does.
7. **Re-estimate when assumptions change.** An estimate built on "10% media" is invalid the
   moment product decides every post gets a thumbnail.
