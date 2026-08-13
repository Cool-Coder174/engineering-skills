# Reference Architectures

**Sources:** *System Design Interview* Ch. 4–15 (Xu); *Grokking the System Design
Interview*, design problems.

A catalog of recurring patterns. **These are patterns, not answers.** The value is
recognizing that your problem is an instance of a solved shape — then re-deriving the
specifics from your own requirements and numbers, because the right design at 1,000 users
is genuinely different from the right design at 100 million.

Each entry gives: the shape, the key decision, the standard solution, and the failure mode
that catches people.

---

## 1. Read-heavy lookup (URL shortener, key-value API, feature flags)

**Shape:** Very high read:write ratio (often 100:1 or higher). Small records. Lookup by key.

**Key decision:** how the key is generated, and where reads are served from.

**Standard solution:** base-62 encode a globally unique ID, or hash the input and resolve
collisions. Writes go to the primary; reads are served from a cache with read replicas
behind it. The hot working set is usually small enough to fit in memory.

**Watch for:**
- **The write path is almost always trivial** and the read path is everything. Size them
  separately — see the worked example in `02-estimation.md`.
- Hash-based schemes need a collision strategy; ID-based schemes need distributed ID
  generation (`04-building-blocks.md` §9).
- 301 vs. 302 redirects: 301 is cached by the browser and removes future traffic entirely
  but destroys your click analytics. This is a product decision disguised as a status code.
- Expiration and cleanup are usually forgotten until storage becomes a problem.

---

## 2. Fan-out (news feed, timelines, activity streams)

**Shape:** Producers create items; many consumers read a merged, ordered view.

**Key decision:** **fan-out on write vs. fan-out on read.** This is the canonical
architectural trade-off and it recurs constantly outside of feeds.

| | Fan-out on write (push) | Fan-out on read (pull) |
|---|---|---|
| On write | Push the item into every follower's precomputed feed | Just store the item |
| On read | Read the precomputed feed — fast | Merge from all followed producers — slow |
| Good for | Read-heavy, bounded follower counts | Write-heavy, unbounded follower counts |
| Fails when | One producer has 50M followers — a single write causes 50M writes | Active users read constantly and each read is expensive |

**Standard solution: hybrid.** Push for ordinary accounts; pull for high-follower
("celebrity") accounts, merged at read time. Pick the threshold from your own follower
distribution, not from a blog post.

**Watch for:**
- **The celebrity problem is the whole problem.** A design that ignores the tail of the
  follower distribution will work in testing and fall over in production.
- Fan-out must be asynchronous and idempotent — it is at-least-once by nature.
- Feed storage grows as O(followers × items). Cap feed length and page beyond it from the
  source.
- Deletes and privacy changes must propagate to every precomputed copy — this is the part
  that is always underspecified, and it is a correctness and compliance issue.

---

## 3. Real-time bidirectional messaging (chat, presence, collaboration)

**Shape:** Low-latency, bidirectional, stateful connections. Delivery guarantees matter.

**Key decision:** the transport, and where connection state lives.

**Standard solution:** WebSocket for the bidirectional path (see `04-building-blocks.md`
§10 — use the weakest transport that meets the requirement). A service discovery layer maps
users to the chat server holding their connection. Messages persist in a store keyed by
`(channel_id, message_id)` where the message ID is time-sortable, giving ordering and
efficient range reads for history. Wide-column stores fit this shape well.

**Watch for:**
- **Persistent connections make everything harder.** Load balancing needs connection
  awareness, deploys drop connections, and **reconnect storms after a deploy are a real load
  event** — plan the reconnect backoff and jitter before the first deploy, not after.
- Message ordering must come from a sequence per channel, not from wall-clock timestamps
  across servers (hazard H-20).
- Offline delivery, read receipts, and presence each need their own storage and each is more
  expensive than it looks. Presence fan-out to large groups is its own scaling problem —
  consider fetching presence on demand rather than broadcasting it.
- End-to-end encryption, if required, constrains every server-side feature (search, moderation,
  server-side history). Decide this at design time; it cannot be added later.

---

## 4. Asynchronous notification delivery (push, email, SMS)

**Shape:** Events trigger messages through third-party providers with variable reliability.

**Standard solution:** event → notification service → per-channel queue → workers → third-party
provider. Queues decouple the trigger from provider latency and outages.

**Watch for:**
- **Third-party providers fail, rate-limit, and are slow.** Every call needs a timeout, a
  jittered retry, and a circuit breaker (hazards H-14, H-15, H-16).
- **Retries duplicate notifications**, which users perceive as a serious bug. Use a
  deduplication key per (user, event) with a retention window.
- Opt-out and preference checks must happen at send time, not at enqueue time — preferences
  change while a message sits in the queue, and sending after opt-out is a compliance issue.
- Templating, localization, and rate limiting per user (nobody wants 40 pushes) belong in
  the service, not scattered across producers.

---

## 5. Crawling and large-scale ingestion

**Shape:** A frontier of work items, politeness constraints, and unbounded scale.

**Standard solution:** URL frontier with priority and per-host queues → fetchers → parser →
content-seen dedup (hash the content, not the URL) → storage → URL extraction → filter →
back to the frontier.

**Watch for:**
- **Politeness is a hard requirement**, not a nicety: one queue per host, one worker per
  queue, with a delay. Getting this wrong gets you blocked and is genuinely rude.
- Content duplication is enormous; dedupe by content hash.
- Traps: infinite URL spaces, redirect loops, enormous files. Bound everything — depth, size,
  time, and redirect count.
- The frontier is the state of the entire system and must be durable and resumable.

---

## 6. Large file storage and sync (Drive, Dropbox)

**Shape:** Large blobs, versioning, multi-device sync, conflicting edits.

**Standard solution:** split files into **blocks**; hash each block; store blocks in object
storage; store metadata (file tree, versions, block lists) in a relational database. Sync
transfers only changed blocks (**delta sync**), with compression and deduplication.

**Watch for:**
- **Block-level delta sync is the whole optimization.** Uploading a 1 GB file because one
  byte changed is the naive design and the difference is orders of magnitude.
- Metadata and blob storage are separate systems, so they can diverge. Design the
  reconciliation and garbage-collection paths explicitly.
- **Conflict resolution needs a stated policy** — last-writer-wins loses data silently
  (hazard H-27). Keeping both versions and letting the user resolve is usually correct.
- Notification of changes to other devices needs long polling or WebSockets, plus a cursor
  so a device that was offline can catch up without a full resync.

---

## 7. Prefix search and autocomplete

**Shape:** Extremely read-heavy, sub-100 ms latency, approximate results acceptable.

**Standard solution:** a **trie** with the top-k results precomputed and cached at each node,
rebuilt offline from aggregated query logs on a schedule (hourly or daily). Serve from
memory, sharded by prefix, fronted by a cache and a CDN for common prefixes.

**Watch for:**
- **The trie is a derived, rebuildable structure** — never a source of truth. Rebuild it
  offline; do not update it synchronously on the write path.
- Freshness is a deliberate trade. Daily rebuilds are fine for most terms and wrong for
  trending events, which usually justifies a separate real-time path for a small hot set.
- Sharding by first letter distributes terribly (English is not uniform). Shard on estimated
  load per prefix range instead.
- Filtering (profanity, safety, personalization) happens at query time and costs latency —
  budget for it.

---

## 8. Video / large media platforms

**Shape:** Enormous storage and egress, expensive transcoding, global distribution.

**Standard solution:** upload to object storage → transcoding pipeline (a DAG of jobs
producing multiple resolutions and formats) → CDN distribution → adaptive bitrate streaming.
Metadata lives in a database; the media never does.

**Watch for:**
- **Egress and storage cost dominate the design**, more than any latency concern. Do the
  bandwidth arithmetic first; it will change your architecture.
- Transcoding is the expensive, failure-prone, long-running part. Make it a resumable
  pipeline with per-stage checkpoints, not one long job.
- Pre-generating every format for every video wastes enormous money on the long tail. Consider
  on-demand transcoding for rarely-viewed content.
- CDN configuration, not the origin, determines the user experience.

---

## 9. Rate limiting and quota enforcement

**Shape:** Distributed counters with a latency budget on the request path.

**Standard solution:** token bucket in a shared store (Redis), keyed per user, per endpoint,
and globally. See `04-building-blocks.md` §8 for the algorithm comparison.

**Watch for:**
- **Local counters do not work across a fleet** — each server enforces its own limit and the
  effective limit is N times what you configured.
- The counter store is on the critical path. Decide the fail-open vs. fail-closed behavior
  explicitly when it is unavailable; both are defensible and the default is usually wrong.
- Race conditions between check and decrement need an atomic operation, not read-then-write
  (hazard H-01).
- Always return `429` with `Retry-After` and `X-RateLimit-*` headers. A client that cannot
  tell it is being limited will retry immediately and make it worse.

---

## 10. Distributed unique ID generation

**Shape:** Unique, ideally sortable IDs without central coordination.

**Standard solution:** Snowflake-style — timestamp + machine ID + sequence. See
`04-building-blocks.md` §9.

**Watch for:** clock skew and backwards jumps (hazard H-20, H-21), machine-ID assignment and
reuse, and the sequence overflowing within a single millisecond under burst.

---

## Using this catalog

1. **Identify the shape**, not the product. "Design Instagram" is fan-out plus media storage.
   "Design a dashboard" is usually read-heavy lookup plus a derived analytical store.
2. **Take the key decision and the failure mode**, then re-derive the rest from your own
   requirements and numbers. Copying an architecture wholesale imports scale you do not have
   — that is the over-engineering failure in `06-simplicity-and-design-laws.md`.
3. **Run the hazard scan** from `data-systems-design/references/hazard-catalog.md`. These
   patterns describe structure; the hazard catalog is what keeps them correct under
   concurrency and failure.
4. **Check the numbers.** Most of these patterns only become necessary above a threshold.
   Below it, the simple version is not just adequate — it is better.
