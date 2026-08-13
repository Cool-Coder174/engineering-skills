# Estimate the Load

**Sources:** *System Design Interview*, Chapter 2 (Xu). *Grokking the System Design
Interview*, Step 3. The latency numbers come from Jeff Dean at Google.

Estimation changes a diagram into a design. It answers two questions. Do you need one
machine or one hundred? Do you need one database or many? The numbers are also the only
honest way to reject a design that is too large.

---

## 1. Powers of two

Every calculation reduces to these units.

| Power | Approximate value | Name | Short form |
|---|---|---|---|
| 2^10 | 1 thousand | 1 Kilobyte | 1 KB |
| 2^20 | 1 million | 1 Megabyte | 1 MB |
| 2^30 | 1 billion | 1 Gigabyte | 1 GB |
| 2^40 | 1 trillion | 1 Terabyte | 1 TB |
| 2^50 | 1 quadrillion | 1 Petabyte | 1 PB |

One byte is 8 bits. One ASCII character is 1 byte.

**Learn these constants:**

- 1 day = 86,400 seconds. Use **10^5 seconds**. The approximation is accurate enough, and it
  makes the arithmetic simple.
- 1 month = about 2.5 x 10^6 seconds
- 1 year = about 3.15 x 10^7 seconds, or 365 days
- 1 million requests each day = about **12 QPS**
- 100 million requests each day = about 1,160 QPS

---

## 2. Latency numbers

These are Jeff Dean's numbers. Hardware improves, so some values are old. The **relative
sizes** are what matter, and those have not changed.

| Operation | Time |
|---|---|
| L1 cache reference | 0.5 ns |
| Branch mispredict | 5 ns |
| L2 cache reference | 7 ns |
| Mutex lock and unlock | 100 ns |
| Main memory reference | 100 ns |
| Compress 1 KB with Zippy | 10,000 ns = 10 us |
| Send 2 KB over a 1 Gbps network | 20,000 ns = 20 us |
| Read 1 MB in sequence from memory | 250,000 ns = 250 us |
| Round trip inside one datacenter | 500,000 ns = 500 us |
| Disk seek | 10,000,000 ns = 10 ms |
| Read 1 MB in sequence from the network | 10,000,000 ns = 10 ms |
| Read 1 MB in sequence from disk | 30,000,000 ns = 30 ms |
| Send a packet from California to the Netherlands and back | 150,000,000 ns = 150 ms |

Units: 1 ns = 10^-9 s. 1 us = 1,000 ns. 1 ms = 1,000 us = 1,000,000 ns.

**These numbers produce six design rules:**

1. **Memory is fast. Disk is slow.** The difference is about 100 times for sequential reads.
   It is much larger for seeks.
2. **Avoid disk seeks.** One seek costs about the same as one sequential megabyte read.
3. **Simple compression is fast.** It costs 10 us for each kilobyte.
4. **Compress data before you send it over a network.** Compression is cheap. Transfer is
   not.
5. **Traffic between regions is expensive.** A round trip between continents costs about
   150 ms. A design with several sequential cross-region calls cannot meet a 200 ms p99
   target. Faster code does not change this result.
6. **Every network call on a request path is a decision.** One round trip inside a datacenter
   costs 500 us. That is about 5,000 times one main memory reference.

---

## 3. Availability numbers

Availability uses a count of nines. A service level agreement (SLA) states the uptime that
you commit to. Large cloud providers set their SLA at 99.9% or higher.

| Availability | Downtime each day | Downtime each week | Downtime each month | Downtime each year |
|---|---|---|---|---|
| 99% | 14.40 min | 1.68 h | 7.31 h | 3.65 days |
| 99.9% | 1.44 min | 10.08 min | 43.83 min | 8.77 h |
| 99.99% | 8.64 s | 1.01 min | 4.38 min | 52.60 min |
| 99.999% | 864 ms | 6.05 s | 26.30 s | 5.26 min |
| 99.9999% | 86.4 ms | 604.8 ms | 2.63 s | 31.56 s |

**Three rules follow from this table:**

1. **Dependencies in series multiply.** A request calls four services. Each service has
   99.9% availability. The limit is 0.999^4, which is about **99.6%**. That is worse than any
   single component. This calculation is the reason to reduce the number of calls, to add
   caches, and to design a degraded mode.

2. **99.99% permits about 4.4 minutes each month.** Deploys, migrations, and failovers
   consume that budget. Outages are not the only cost. Suppose a deploy causes 30 seconds of
   unavailability, and you deploy every day. You already spent 15 minutes.

3. **Select the target from the business need. Then check that the architecture can reach
   it.** Three nines with a manual failover procedure is not three nines.

---

## 4. Formulas

### QPS
```
Daily active users = monthly active users x (fraction that is active each day)
Actions each day   = daily active users x actions for each user each day
Average QPS        = actions each day / 86,400        (use / 10^5)
Peak QPS           = average QPS x peak factor
```
Use 2 as the default peak factor. Use a value from 2 to 10. Use a higher value when the load
changes through the day, or when events drive the load.

### Storage
```
Storage each day       = writes each day x average record size
Storage at the limit   = storage each day x 365 x retention years
With replication       = x replication factor (3 is typical)
With overhead          = x 1.2 to 1.5 (indexes, metadata, fragmentation, free space)
```

Calculate the **largest** term first. Suppose 10% of records hold 1 MB of media, and the rest
hold 200 bytes of text. Then the media term is the whole answer. Calculate it and continue.

### Bandwidth
```
Inbound  = write QPS x average request size
Outbound = read QPS x average response size
```
Outbound is usually the larger number. It is usually the larger cost. For a system with much
media, outbound bandwidth is the main reason to add a CDN.

### Memory and cache
```
Working set = requests each day x fraction of distinct hot items x item size
Cache size  = working set x 1.2   (overhead)
```

Start with this rule: about 20% of items produce about 80% of the traffic. So a cache that
holds the hot 20% gives most of the benefit. Check this rule against real access data when
you have it. The real distribution is often much more uneven.

### Server count
```
Servers = peak QPS / QPS for each server
```

Typical capacity for each server, as an order of magnitude only. Measure your own values.

- Simple API on current hardware: 500 to 1,000 QPS
- Heavy computation: 50 to 200 QPS
- Static content: much higher

Then add capacity for failure (N+1 or N+2). Then add capacity for the highest peak.

### Concurrent requests
```
Concurrent requests = QPS x average request duration      (Little's Law)
```

**Little's Law is the most useful formula on this page.** An example: 1,000 QPS with a
200 ms average duration gives **200 requests in progress**. That number sizes your connection
pools and thread pools.

The same formula shows a failure mode. A dependency slows from 200 ms to 2 s. The number of
requests in progress becomes 10 times larger. The pool runs out.

---

## 5. Worked example: Xu's Twitter estimate

**Assumptions.** Record these first. A reviewer must be able to challenge them.

- 300 million monthly active users
- 50% of users use the product each day
- 2 posts for each user each day, on average
- 10% of posts contain media
- Media averages 1 MB
- We keep data for 5 years

**QPS**
```
Daily active users = 300M x 50%                = 150M
Post QPS           = 150M x 2 / 86,400         = about 3,500 QPS
Peak QPS           = 3,500 x 2                 = about 7,000 QPS
```

**Storage. Media is the largest term.**
```
Average post: post_id 64 B + text 140 B + media 1 MB
Media each day = 150M x 2 x 10% x 1 MB         = 30 TB each day
After 5 years  = 30 TB x 365 x 5               = about 55 PB
```

**Two conclusions follow at once.**

1. 7,000 peak writes each second is more than one database can accept. So the design needs
   partitioning.
2. 55 PB of media does not belong in a database. It belongs in object storage. The database
   holds only the references.

Arithmetic produced both conclusions. Without the arithmetic, both would be guesses.

---

## 6. Second worked example: a read-heavy system

This example shows how far reads and writes can differ.

**Assumptions.** A URL shortener. 100 million new URLs each month. 100 reads for each write.
5-year retention. 500 bytes for each record.

```
Write QPS   = 100M / (30 x 86,400)             = about 40 QPS
Read QPS    = 40 x 100                         = 4,000 QPS
Records     = 100M x 12 x 5                    = 6 billion
Storage     = 6B x 500 B                       = 3 TB
Hot data    = 4,000 x 86,400 x 20% x 500 B     = about 35 GB each day
```

**Three conclusions follow.**

1. Writes are small. One modest server handles 40 QPS.
2. Reads are 100 times the writes.
3. The hot data fits in memory on a single cache node.

So the design is read replicas plus a cache. The system needs no partitioning for years.

**A design that proposed partitioning here would be too large. The numbers prove it.**

---

## 7. Rules for good estimation

1. **Round the numbers.** Use 10^5 for one day. Precision is not the goal. It costs the time
   that you need for design.
2. **Write each assumption down.** Every estimate depends on assumptions. A reviewer must
   attack the assumption, not the arithmetic.
3. **Write the unit after every number.** The value "5" is unclear. The value "5 MB" is
   clear. Unit confusion is the most frequent estimation error.
4. **Calculate the largest term first.** One term is usually 100 times the others. Find it.
   The other terms do not change the answer.
5. **Compare the result against reality.** Suppose your estimate needs 40,000 servers. Then
   either the estimate or the design is wrong.
6. **Prefer measurement.** In real work, production metrics are better than estimates.
   Estimate only what does not exist yet. Measure everything that exists.
7. **Estimate again when an assumption changes.** An estimate that assumes "10% media"
   becomes invalid when the product team adds a thumbnail to every post.
