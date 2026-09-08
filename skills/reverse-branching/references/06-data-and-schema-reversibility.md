# Data and Schema Reversibility

**Sources:** *Site Reliability Engineering* (Beyer, Jones, Petoff, Murphy), Ch. 26, "Data
Integrity: What You Read Is What You Wrote", p. 412–447. *Release It!* (Nygard), Sec. 18.4,
"Zero Downtime Deployments", p. 331–334.

This is the hard half of reverse branching. A code revert is cheap. A data revert is not.
Read this file when a change touches a table, a row, a file, or a delete path. **Word
discipline.** A **revert** undoes code or configuration. A **restore** brings data back from
a copy. A **validator** verifies an invariant out of band.

---

## 1. The goal is availability, not the backup

**No one wants a backup. People want a restore.** SRE Ch. 26, p. 416. Making a backup yields
no visible benefit, so teams neglect the task. **Data integrity is the means. Data
availability is the goal.** p. 418. Google SRE does not ask a team to practice its backups.
The team defines a service level objective for data availability under named failure modes,
then demonstrates that it meets it. p. 419.

**War story — the provider that lost nothing.** SRE Ch. 26, p. 418–419 describes an email
provider that suffered a data outage. For 10 days users worked with temporary accounts. The
provider then announced that the old mail and contacts were gone, and furious users left.
Several days later it announced that the data was recoverable after all. No data had been
lost. The data had only been unreachable for too long. To a user, that equals loss.

**How long is too long?** A 2011 Gmail incident showed four days to be a long time, and
perhaps too long. Google Apps uses 24 hours as a starting point. p. 412. Your own threshold is
a local decision. Uptime and integrity also carry separate requirements. An objective of
99.99% uptime allows about one hour of downtime a year. An objective of 99.99% good bytes in
a 2 GB artifact garbles up to 200 KB. That result is catastrophic. p. 413.

---

## 2. A backup is not an archive

| | Backup | Archive |
|---|---|---|
| Purpose | Recovery after a fault | Auditing, discovery, compliance |
| An application can load it | Yes | No |
| Acceptable restore time | Inside the uptime needs of the service | A week during a month-long audit |
| Example | An hourly copy beside the live store | Monthly audit logs offsite, kept seven years |

**The property that matters is loadability.** A backup loads back into the application. An
archive does not. A plan that names an archive as the restore path cannot meet an uptime
requirement. **The signal:** the plan names a long-term store, and no engineer can say which
job loads the data back into the service. SRE Ch. 26, p. 416–417. Hazard `R-32` in
`rollback-hazard-catalog.md`.

---

## 3. The 24 combinations of data loss

SRE Ch. 26, p. 420 counts 24 distinct failure types from three factors in any combination.

| Factor | Values the book lists | Count |
|---|---|---|
| Cause | User action, operator error, an application bug, a defect in infrastructure, faulty hardware, a site catastrophe | 6 |
| Scope | Wide, across many entities. Narrow, against a small subset of users. | 2 |
| Rate | Big bang. Creeping. | 2 |

A big bang replaces 1 million rows with 10 rows in one minute. A creeping loss deletes 10
rows every minute across weeks. **A plan must cover any combination.** A strategy that stops
a creeping bug helps nothing when the datacenter burns. p. 420.

**Which combination actually happens?** A study of 19 data recovery efforts at Google names
the two most common user-visible losses. They are data deletion and a loss of referential
integrity. Software bugs caused both. The hardest variants were low-grade corruption found
weeks to months after the release. p. 421. That shape may need one point in time per artifact.
Section 6 covers it.

---

## 4. Replication is not recoverability

**Replication and redundancy are not recoverability.** SRE Ch. 26, p. 421. This is the most
common defect in a data reversal plan. A datastore that syncs replicas automatically
guarantees one thing. A corrupt row or an errant delete reaches every copy, and it usually
arrives before you can isolate the problem. p. 422.

**A nonserving copy in another format helps only partly.** A frequent database export to a
native file protects against user error and application bugs. It does nothing against a fault
in a lower layer. **Diversity is the property that matters.** To protect against a fault at
layer X, store the data on diverse components at that layer. A defect in a disk device driver
is unlikely to affect tape drives. p. 422.

**The cost of depth is freshness.** SRE Ch. 26, p. 422 gives these figures.

| Copy layer | Time to make the copy | Recent data you can lose |
|---|---|---|
| A database transaction to a replica | Seconds | Near zero, and it copies the error too |
| A database snapshot exported to the filesystem | About 40 minutes | Up to 40 minutes |
| A full backup of the underlying filesystem | Hours | Hours of transactions |

A restore takes as long as the copy, and the freshest copy may not survive the fault. `R-28`.

---

## 5. Defense in depth: the three layers

SRE Ch. 26, p. 423–424 states that no single mechanism guards against the 24 combinations.
Each successive layer covers progressively rarer scenarios.

| Layer | Mechanism | Primary defense against | Secondary defense against |
|---|---|---|---|
| 1 | A trash folder that the user controls | User error | — |
| 1 | Soft deletion | Developer error | User error |
| 1 | Lazy deletion, inside the storage layer | Internal developer error | External developer error |
| 2 | Backups and their recovery methods | A bad release, a wide fault, a site fault | — |
| 3 | Regular out-of-band validation | Low-grade corruption found late | — |
| Overarching | Replication | Site loss and latency | Nothing you may plan on |

Sources: SRE Ch. 26, p. 423–426. The summary of layer 1 appears on p. 426.

### 5.1 Layer 1 — soft deletion

**Soft deletion marks data as deleted at once, and only administrative code paths can then
use it.** SRE Ch. 26, p. 424. Those paths include legal discovery, recovery of a hijacked
account, enterprise administration, user support, and troubleshooting. Apply soft deletion
when the user empties the trash. Give an administrator a tool that undeletes an item.

**Common delays are 15, 30, 45, or 60 days.** Gmail lets a user reach messages deleted fewer
than 30 days ago. In Google's experience most account hijacking and data integrity issues are
reported or detected within 60 days. The book therefore states that the case for a longer
delay may not be strong. p. 425. Keep that hedge. The exact delay depends on your policy, the
applicable law, storage cost, and the price of the product.

**War story — who writes the deletion bug.** SRE Ch. 26, p. 425 names the source of the most
devastating acute deletion cases at Google. They came from application developers who did not
know the existing code and who worked on deletion code. Batch processing pipelines, such as an
offline MapReduce job, were the worst case. The design consequence is direct. Build the interface so that a
developer who does not know the code cannot circumvent soft deletion. A storage offering with
built-in soft deletion and undeletion APIs achieves this. Then enable it.

**Lazy deletion is the same idea one layer down.** The storage system purges the data out of
sight of the application. The provider keeps it for up to a few weeks. A long delay costs too much where
data is short-lived, and it is impractical where a privacy guarantee requires destruction
inside a set time. Revision history is not a substitute, because some implementations remove
the previous states on a deletion. p. 426. Hazard `R-30`.

### 5.2 Layer 2 — backups and recovery

**Backups do not matter. Recovery matters.** SRE Ch. 26, p. 426–427. The recovery scenarios
drive the backup decisions. The reverse order is the defect. The scenarios decide four
things. Which methods you use. How often you establish a restore point. Where you store the
copies. How long you keep them. p. 427.

Two questions shape all four. How much recent data can you lose? How fast must users return
to service? p. 427. **Modern:** the industry names these answers the Recovery Point Objective
and the Recovery Time Objective. The book does not. Write both numbers in the record.

**The tiered strategy.** SRE Ch. 26, p. 428–429.

| Tier | Where it lives | Retention | Restore time | Protects against |
|---|---|---|---|---|
| 1 | Closest to the live datastore, on the same or similar storage technology | Hours to single-digit days | Minutes | Most software bugs and developer error |
| 2 | A random access distributed filesystem local to the site | Single to low double-digit days | Hours | A fault in a serving storage technology, and a bug found too late for tier 1 |
| 3 | Nearline tape libraries and offsite media | Longer, and taken less often | Hours to days | A site fault, such as a power outage or filesystem corruption |

**Set tier 2 retention from your release cadence.** If you push new code twice a week, keep
these backups at least a week or two. p. 428.

**Set the outer window from your detection latency.** Low-grade mutation or deletion bugs
demand the longest window. Engineers noticed some of them months after the loss began. High velocity works against that, because code and schema changes can render an old
backup impossible to use. Google chose to draw the line between 30 and 90 days for many
services. p. 428. Hazards `R-31` and `R-29`.

**Scale changes the method.** One task that verifies 700 petabytes at 300 MB/s takes 8
decades. Verify data after it becomes immutable, then take incremental copies. That choice
gives a restore of more than 1,000 dependent backups in series, so shard the data and run N
tasks in parallel. p. 429–430.

### 5.3 Layer 3 — early detection

**Bad data propagates.** References to missing or corrupt data are copied. Links multiply.
The sooner you learn about a loss, the easier and more complete the recovery. p. 431.

**Validate out of band.** A validation pipeline runs outside the serving path and verifies
invariants within and between the datastores. Gmail developers know that a bug which
introduces an inconsistency in production data is detected inside 24 hours. That knowledge
lets them change the production storage implementation more often than once a week. p. 432. **Validate only the invariants whose failure devastates users.** A validator that is
too strict fails on appropriate changes, and engineers then abandon validation. A validator
that is not strict enough lets corruption pass. p. 433.

**War story — Google Drive, 2013.** Drive validates periodically that file contents align
with the listings in Drive folders. A mismatch means that some files lack data. The
infrastructure developers also made the validators repair such inconsistencies automatically.
In 2013 that work turned an emergency about disappearing files into routine work that the
team fixed on the next working day. p. 433.

**A validator may repair. A validator must not repair by deleting user data.** SRE Ch. 17,
p. 242 states that the value of such a tool lies in reporting the problem. The same page
states that the tool should avoid remediation on its own by deleting a large amount of user
data. Hazard `R-38`
and `07-automation-safety.md`.

**Validation costs capacity.** Validators lower the server-side cache hit rate, so Gmail
exposes rate-limit controls for them. p. 433. Validation also demands job management,
monitoring, playbooks, and validation APIs, so a central team supplies the framework. p. 434.

**Alert on the global deletion rate.** Per-user alerts are less useful, because per-user rates
vary naturally. Alert when the deletion rate aggregated across all users crosses an extreme
threshold, such as 10 times the observed 95th percentile. p. 447, fn. 133. Hazard `R-33`.

---

## 6. Point-in-time recovery

A recovery from low-grade corruption must restore different subsets of data to different
restore points. The book calls this point-in-time recovery outside Google, and time-travel
inside Google. SRE Ch. 26, p. 421.

**Keep the book's hedge.** p. 421 calls one thing a chimera today. That thing is a backup and
recovery solution with two properties. It provides point-in-time recovery across ACID and BASE
datastores. It also meets strict uptime, latency, scalability, velocity, and cost goals. Many
projects
therefore compromise with a tiered strategy and no point-in-time recovery. **The rule the book gives.**
Trade point-in-time recovery against the tiered strategy if you must. Do not choose neither.
If you can have both, use both. p. 421.

**A deep restore merges with live data.** A snapshot from far in the past must merge with the
current state. That merge complicates the restore. p. 423. Budget it in the Recovery Time
Objective.

---

## 7. Recovery that no test exercised does not exist

**When does a light bulb break?** SRE Ch. 26, p. 434–435 uses the question to make the point.
Often the bulb had already failed, and you learn about the failure at the unresponsive flick
of the switch. Your recovery dependencies can sit in a latent broken state that you discover
only when you try to recover. Two actions detect that state. p. 435.

1. Test the recovery process continuously, as part of normal operations.
2. Alert when a recovery run gives no heartbeat that indicates success.

**Only a full end-to-end test earns confidence.** Even after a recent successful recovery,
parts of the process can break. The book states the lesson of the chapter plainly. You know
that you can recover your recent state only when you actually do so. A manual, staged test
becomes drudgery, so nobody runs it often enough. Automate it. Confirm five items. p. 435.

1. Are the backups valid and complete, or are they empty?
2. Do you have enough machine resources for the setup, the restore, and the post-processing?
3. Does the recovery finish in reasonable wall time?
4. Can you monitor the state of the recovery while it runs?
5. Are you free of critical dependencies outside your control, such as an offsite media vault
   that is not available 24 hours a day?

Google's own tests found each of these failures. p. 435–436. Hazard `R-29`. The Reversal
Record holds the rehearsal date, and "never" blocks the merge for a data-class change.

---

## 8. Expand, migrate, contract

This is the reversible shape for a schema change. Release It! Sec. 18.4 names the three
phases expansion, rollout, and cleanup, p. 331–334. This skill and `data-systems-design` name
the same shape expand, migrate, contract.
`data-systems-design/references/04-encoding-and-evolution.md`, Section 4, owns the
compatibility mechanics. It holds the backfill rules, the locking rules, and the tag-reuse
rules. This section owns only the reversibility consequence of each phase. **Why the shape exists.** During a deploy some
servers run version N and some run version N+1. Both versions are present at the same time,
and every external reference is then an opportunity for a version conflict. p. 331.

| Phase | The change it makes | Is a code revert safe? | The rule and its source |
|---|---|---|---|
| Expand | Adds a table, a nullable column, a new endpoint name, a versioned asset URL | Yes | Add every future NOT NULL column as nullable. Add no referential integrity rule yet. p. 333 |
| Migrate | Backfills, writes both shapes, moves the reads | Yes, while both shapes stay current | Bridging triggers populate the new shape from old writes, and the old shape from new writes. p. 333 |
| Contract | Drops a column, adds NOT NULL, adds a foreign key, deletes old assets | No | This is the point of no return. Run it only after the release bakes. p. 334 |

**Expansion, in detail.** Release It! Sec. 18.4, p. 332–333.

1. Give each revision of a static asset its own URL, such as `/static/1.1/styles.css`.
2. Give each revision of a web service its own endpoint name.
3. Put a version identifier inside a socket protocol. Update the receiver before the sender.
   As an alternative, define separate service pools in the load balancer on other ports.
4. Expect the database to produce the most conflicts and the worst ones.
5. Name every column and value explicitly in each INSERT statement and UPDATE statement. A
   `SELECT *` is unlikely to help. Apply the same rule to reporting queries.

**Rollout, in detail.** The deploy of the new code should be trivial after expansion. It can
take a few hours to a few days. Let a couple of servers bake on the new code base for a day
or more. Two service pools in the load balancer keep session failover inside one version.
p. 333–334. For the exposure ladder, read `03-progressive-rollout-and-canary.md`.

**Cleanup, in detail.** Run it only after the new release has baked long enough for the team
to accept it. Remove the bridging triggers and the extra service pools. Drop the columns and
tables that nothing uses. Convert the columns that need it to NOT NULL. Add the referential
integrity relations. p. 334. The book adds a caveat. Constraints enforced in the database can
cause large problems for the ORM layer. **Never place expand and contract in one release.**
Hazard `R-26`.

---

## 9. Why a code revert past a migration corrupts data

**The old version reads data that the new version wrote as corrupt or incomplete.**
Release It! Sec. 18.4, p. 333 states the mechanism. The sequence has four steps.

1. The new version writes rows in the new shape.
2. You revert the code to the old version.
3. The old version reads only the old shape.
4. The rows that the new version wrote carry no old shape, so the old version treats them as
   missing data or as wrong data.

**The bridging trigger is what makes the revert safe.** It populates the new structure from an
old write. It also populates the old structure from a new write. It runs one row at a time. It
does the same work as the forward migration script. p. 333. Remove the triggers in the
cleanup phase, and not before.

**A constraint added during expansion breaks the revert a second way.** Add a NOT NULL
constraint or a referential integrity rule beside the new column, and the old version violates
that constraint at once. The two versions cannot coexist, so no revert is possible. p. 333.
Hazard `R-27`.

**The contract step is the point of no return.** Name it in the Reversal Record. After
contract, the reverse action is a restore, not a revert. Read the restore time from the tier
table in Section 5.2. It is hours, not minutes.

---

## 10. Two war stories about the restore

**Gmail, February 2011 — the restore from GTape.** SRE Ch. 26, p. 436–438. A pager fired late
on a Sunday evening. Gmail had lost a significant amount of user data, and the internal
redundancy could not recover it. This was the first large-scale use of GTape, the offline
tape backup system, on live customer data. It was not the first such restore, because the
team had simulated similar situations many times. That rehearsal bought three results. The
team estimated the time to restore the majority of affected accounts. It restored every
account within several hours of that estimate. It recovered more than 99% of the data before
the estimate.
Defense in depth is the reason the tape existed. Disks and a fast network cannot defend
against a zero-day defect in a device driver or a filesystem.

**Google Music, March 2012 — runaway deletion.** SRE Ch. 26, p. 438–443. A user reported that
playable tracks were skipped. An engineer found that a privacy-protecting deletion pipeline
had removed the reference from the metadata of a track to its audio data. A hasty MapReduce
job measured the damage. The refactored pipeline had removed about 600,000 audio references
and affected 21,000 users, and its first run had happened over a month earlier. The tapes
lived offsite, so the recovery needed 5,475 restore jobs and the truck delivery of 5,337
tapes. A vendor omitted 1,862 of them, and 17 tapes were bad. Only 436,223 tracks existed on
tape, because the pipeline had eaten the rest before any backup captured them. Restoring 1.5
petabytes took seven days. The root cause was a race condition between two pipeline stages
designed to run three hours apart. As the data grew, the upstream stages needed more time,
engineers applied performance optimizations, and the probability of the race increased. The
Music restore was slow for three reasons. Nothing had rehearsed a restore at that size. The
media was offsite. The loss predated the retention of the fresh tiers.

---

## 11. Using this file

### 11.1 What it writes into the Reversal Record

Answer the data half of the record with seven values. The tables and objects that revert,
plus their restore layer from Section 5. The Recovery Point Objective and the Recovery Time
Objective from Section 5.2. The point of no return, named by migration file, from Section 8.
The date of the last full end-to-end restore test, from Section 7. The validator, its period,
and the deletion rate alert, from Section 5.3. The residual risk, which is the window between
the loss and the oldest usable restore point.

### 11.2 Hazard index

Run these entries from `rollback-hazard-catalog.md` against any change that touches data.

| Hazard | Short name | Section |
|---|---|---|
| 🔴 `R-26` | A code revert across a destructive migration | 8, 9 |
| 🔴 `R-27` | A constraint added during expansion | 8, 9 |
| 🔴 `R-28` | Replication treated as the backup | 4 |
| 🔴 `R-29` | A restore that was never run | 7 |
| 🔴 `R-30` | A hard delete with no soft delete | 5.1 |
| 🟡 `R-31` | A retention window shorter than the detection window | 5.2 |
| 🟡 `R-32` | An archive presented as a backup | 2 |
| 🟡 `R-33` | No alert on the deletion rate | 5.3 |
| 🟡 `R-38` | Automated repair that deletes data | 5.3 |

### 11.3 Proportionality

| The change | What this file requires |
|---|---|
| A read-only query or a report | Nothing |
| A new nullable column, no backfill, no constraint | The expand row of the table in Section 8 |
| A backfill or a bulk update | Sections 5.2, 7, and 8. A rehearsal date. |
| A new delete path, or an edit to one | Sections 5.1 and 5.3. A soft delete and a rate alert. |
| A destructive migration | Every section. A named point of no return and a measured restore time. |
| A change to a bulk deletion pipeline | Every section, plus group G of the catalog |

"Not applicable" is a valid result. A change that adds no write path and no delete path needs
nothing from this file.

### 11.4 Related reading

| File | What it owns |
|---|---|
| `01-the-reversibility-contract.md` | The four questions and the reversibility classes |
| `03-progressive-rollout-and-canary.md` | The exposure ladder and the bake period |
| `04-change-induced-emergency.md` | The order of moves when a change caused the outage |
| `07-automation-safety.md` | Rate limits, idempotence, and the stop control for a bulk job |
| `rollback-hazard-catalog.md` | The full `R-` catalog and its severity rules |
| `data-systems-design/references/04-encoding-and-evolution.md` | Schema compatibility both ways |
| `data-systems-design/references/12-correctness-and-integrity.md` | Auditing and integrity checks |
| `systems-programming/references/02-files-and-directories.md` | Sec. 6 and Sec. 7. Atomic rename, and the durable file update |
