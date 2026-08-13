# Reference Architectures

**Sources:** *System Design Interview*, Chapters 4 to 15 (Xu). *Grokking the System Design
Interview*, design problems.

This is a catalog of patterns that occur often.

**These are patterns. They are not answers.** Use the catalog to identify that your problem
has a known shape. Then derive the details from your own requirements and numbers. The
correct design for 1,000 users differs from the correct design for 100 million users.

Each entry gives four items: the shape, the main decision, the standard solution, and the
failure mode that engineers miss.

---

## 1. Read-heavy lookup

Examples: a URL shortener, a key-value API, a feature-flag service.

**Shape.** Many more reads than writes, often 100 to 1 or higher. Small records. Lookup by
key.

**Main decision.** How to generate the key, and where to serve reads from.

**Standard solution.** Encode a unique ID in base 62, or hash the input and handle collisions.
Writes go to the primary database. Reads come from a cache, with read replicas behind it. The
hot data usually fits in memory.

**Four items to watch:**

1. **The write path is almost always small. The read path is the whole design.** Estimate them
   separately. See the second worked example in `02-estimation.md`.
2. A hash scheme needs a collision plan. An ID scheme needs distributed ID generation. See
   `04-building-blocks.md`, Section 9.
3. **Status 301 compared to status 302.** The browser caches a 301 redirect, which removes
   future traffic and also removes your click data. This is a product decision that looks like
   a status code.
4. Teams forget expiry and cleanup until storage becomes a problem.

---

## 2. Fan-out

Examples: a news feed, a timeline, an activity stream.

**Shape.** Producers create items. Many consumers read one merged, ordered view.

**Main decision.** Fan-out on write, or fan-out on read. **This is the standard architectural
trade-off, and it occurs far outside feeds.**

| Property | Fan-out on write (push) | Fan-out on read (pull) |
|---|---|---|
| On write | Copy the item into the feed of every follower | Store the item once |
| On read | Read a prepared feed. Fast. | Merge items from every producer. Slow. |
| Good for | Many reads, and a limited follower count | Many writes, and an unlimited follower count |
| Fails when | One producer has 50 million followers, so one write causes 50 million writes | Active users read often, and each read is expensive |

**Standard solution: use both.** Push for normal accounts. Pull for accounts with many
followers, and merge at read time. Select the threshold from your own follower distribution.
Do not take it from an article.

**Four items to watch:**

1. **The account with many followers is the whole problem.** A design that ignores the tail of
   the follower distribution works in a test and fails in production.
2. Fan-out must be asynchronous and idempotent, because delivery is at-least-once.
3. Feed storage grows with followers multiplied by items. Limit the feed length. Read older
   items from the source.
4. **Deletes and privacy changes must reach every copy.** Teams always underspecify this part.
   It is both a correctness problem and a compliance problem.

---

## 3. Two-way messaging in real time

Examples: chat, presence, live editing.

**Shape.** Low latency. Data moves in both directions. Connections hold state. Delivery
guarantees matter.

**Main decision.** The transport, and where the connection state lives.

**Standard solution.** Use WebSocket for the two-way path. First check
`04-building-blocks.md`, Section 10, and select the simplest transport that meets the
requirement. A discovery service maps each user to the server that holds the connection.
Messages persist in a store keyed by channel ID and message ID. Make the message ID sortable
by time. That key gives order and efficient reads of history. A wide-column database fits this
shape.

**Four items to watch:**

1. **Connections that hold state make everything harder.** Load balancing needs
   connection-aware routing. A deploy closes every connection. **After a deploy, all clients
   reconnect at once. That reconnection is a real load event.** Plan the reconnect delay and
   its random offset before the first deploy.
2. Message order must come from a sequence for each channel. Do not order messages by wall
   clock across servers. That is hazard H-20.
3. Offline delivery, read receipts, and presence each need storage. Each costs more than it
   appears to. Presence for large groups is its own scaling problem. Consider fetching
   presence on demand instead of sending it to everyone.
4. **End-to-end encryption limits every server-side feature**, including search, moderation,
   and server-side history. Decide this at design time. You cannot add it later.

---

## 4. Asynchronous notification delivery

Examples: push messages, email, SMS.

**Shape.** Events cause messages. Third-party providers deliver them. Provider reliability
varies.

**Standard solution.** An event reaches a notification service. The service writes to a queue
for each channel. Workers read the queue and call the provider. The queue separates the event
from provider latency and provider outages.

**Four items to watch:**

1. **Providers fail, apply rate limits, and run slowly.** Every call needs a timeout, a retry
   with a random delay, and a circuit breaker. See hazards H-14, H-15, and H-16.
2. **A retry sends a duplicate notification.** Users see a duplicate as a serious defect. Use
   a deduplication key for each user and event, with a retention period.
3. Check the user preferences when you send, not when you enqueue. Preferences change while a
   message waits in the queue. Sending after a user opts out is a compliance problem.
4. Put templates, localization, and a per-user rate limit inside the service. Nobody wants 40
   push messages. Do not spread this logic across the producers.

---

## 5. Crawling and large-scale ingestion

**Shape.** A list of work items, limits on politeness, and unlimited scale.

**Standard solution.** A URL frontier holds work items with priorities and one queue for each
host. Fetchers read the frontier. A parser reads the content. A deduplication step hashes the
content. Storage keeps the result. A URL extractor finds new links. A filter removes unwanted
links. New links return to the frontier.

**Four items to watch:**

1. **Politeness is a requirement, not an option.** Use one queue for each host, one worker for
   each queue, and a delay between requests. Sites block impolite crawlers, and impolite
   crawling is rude.
2. Duplicate content is very common. Deduplicate by content hash, not by URL.
3. **Traps exist.** Examples: infinite URL spaces, redirect loops, and very large files. Set a
   limit on depth, size, time, and redirect count.
4. The frontier holds the state of the whole system. It must be durable. It must resume after
   a restart.

---

## 6. Large file storage and sync

Examples: Google Drive, Dropbox.

**Shape.** Large files, versions, sync across devices, and conflicting edits.

**Standard solution.** Divide each file into **blocks**. Hash each block. Store the blocks in
object storage. Store the metadata in a relational database. The metadata holds the file tree,
the versions, and the block list for each version. Sync transfers only the blocks that
changed. Add compression and deduplication.

**Four items to watch:**

1. **Block-level sync is the whole optimization.** Uploading a 1 GB file after a one-byte
   change is the naive design. The difference is very large.
2. The metadata and the object storage are separate systems, so they can diverge. Design the
   reconciliation process and the cleanup process.
3. **Conflicts need a stated policy.** A last-writer-wins policy loses data silently. That is
   hazard H-27. Keeping both versions and asking the user is usually correct.
4. Other devices need to learn about changes. Use long polling or WebSockets. Give each device
   a cursor, so that a device that was offline can catch up without a full resync.

---

## 7. Prefix search and autocomplete

**Shape.** Very many reads. Latency under 100 ms. Approximate results are acceptable.

**Standard solution.** Build a trie. Store the top results at each node. Rebuild the trie
offline from query logs, every hour or every day. Serve it from memory. Partition it by
prefix. Put a cache in front. Put common prefixes in a CDN.

**Four items to watch:**

1. **The trie is a derived structure that you can rebuild. It is never authoritative.** Build
   it offline. Do not update it during a write.
2. Freshness is a deliberate trade. A daily rebuild is correct for most terms and wrong for
   new events. So most systems add a separate real-time path for a small set of hot terms.
3. Partitioning by first letter distributes badly, because letters are not evenly used.
   Partition by the estimated load for each prefix range.
4. Filtering costs latency. This includes filters for offensive terms, safety, and
   personalization. Include that cost in the latency budget.

---

## 8. Video and large media platforms

**Shape.** Very large storage. Very large outbound bandwidth. Expensive transcoding. Global
distribution.

**Standard solution.** Upload to object storage. A transcoding pipeline produces several
resolutions and formats. A CDN distributes the output. Clients use adaptive bitrate streaming.
The database holds the metadata. The database never holds the media.

**Four items to watch:**

1. **Bandwidth cost and storage cost control this design**, more than latency does. Calculate
   the bandwidth first. The result changes the architecture.
2. Transcoding is expensive, slow, and prone to failure. Build it as a pipeline that can
   resume, with a checkpoint after each stage. Do not build it as one long job.
3. Producing every format for every video wastes money on videos that nobody watches. Consider
   transcoding on demand for rare content.
4. The CDN configuration decides the user experience. The origin does not.

---

## 9. Rate limiting and quotas

**Shape.** Distributed counters with a latency budget on the request path.

**Standard solution.** A token bucket in a shared store such as Redis. Use one bucket for each
user, one for each endpoint, and one global bucket. For the algorithm comparison, read
`04-building-blocks.md`, Section 8.

**Four items to watch:**

1. **Local counters do not work across many servers.** Each server enforces its own limit, so
   the real limit is N times the configured limit.
2. The counter store is on the request path. Decide what happens when it is unavailable. Both
   "permit the request" and "reject the request" are defensible. The default is usually wrong.
3. The check and the decrement need one atomic operation. A read followed by a write is hazard
   H-01.
4. Always return status 429 with a `Retry-After` header and `X-RateLimit-*` headers. A client
   that cannot detect the limit retries at once and makes the problem worse.

---

## 10. Distributed unique ID generation

**Shape.** IDs that are unique and, if possible, sortable, with no central coordination.

**Standard solution.** A Snowflake-style ID: a timestamp, a machine ID, and a sequence number.
Read `04-building-blocks.md`, Section 9.

**Three items to watch:** clock skew and clocks that move backward, which are hazards H-20 and
H-21. Machine-ID assignment and reuse. Sequence overflow inside one millisecond during a
burst.

---

## How to use this catalog

1. **Identify the shape, not the product.** "Design Instagram" is fan-out plus media storage.
   "Design a dashboard" is usually read-heavy lookup plus a derived analytical store.

2. **Take the main decision and the failure mode. Derive the rest yourself.** Copying a whole
   architecture imports a scale that you do not have. That is the fault described in
   `06-simplicity-and-design-laws.md`.

3. **Run the hazard scan.** The list is in
   `data-systems-design/references/hazard-catalog.md`. These patterns give structure. The
   hazard list keeps that structure correct during concurrent operation and during failures.

4. **Check the numbers.** Most of these patterns become necessary only above a threshold.
   Below that threshold, the simple version is not merely adequate. It is better.
