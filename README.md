# Engineering Skills

A suite of Agent Skills that make an AI coding agent work like a senior engineer. The skills
enforce one order of work:

1. Clarify the problem before you design it.
2. Size it in numbers before you choose an architecture.
3. Record each design decision with the alternative you rejected.
4. Write the code against robustness invariants.
5. Check the code against the spec, with evidence.
6. Review for the failure modes that appear only under concurrency, failure, attack, and scale.

Grounded in eight books:

| Book | What it contributes |
|---|---|
| **[*Designing Data-Intensive Applications*][ddia]** — Kleppmann | Correctness under concurrency, replication, and failure. The 48-hazard catalog. |
| **[*Cryptography and Network Security*][stallings]** — Stallings | Security services and mechanisms, threat framing, the 64-vulnerability catalog |
| **[*Advanced Programming in the UNIX Environment*][apue]** — Stevens and Rago | System call semantics, atomicity, durable file writes, signals, threads. Most of the 55-failure catalog. |
| **[*Systems Programming*][donovan]** — Donovan | The machine model, the design procedure for a system program, the four loader functions |
| **[*System Design Interview: An Insider's Guide*][xu]** — Alex Xu | The design method, back-of-the-envelope estimation, the scaling ladder |
| **[*Grokking the System Design Interview*][grok]** — Design Gurus | Building-block selection. The seven-step design process. |
| **[*Database Internals*][dbi]** — Alex Petrov | Database selection methodology, storage engine trade-offs, the RUM conjecture |
| **[*Code Simplicity*][cs]** — Max Kanat-Alexander | The laws of software design. The anti-over-engineering discipline. |

---

## Why this exists

Agents are good at producing code that works on the happy path in a single-threaded test.
They are much worse at two things that cause real problems.

**They skip the design process.** They answer before clarifying, propose architectures with
no numbers behind them, and choose Kafka at 40 writes per second. The estimation gate and
the simplicity pass exist to stop that.

**They miss the failure modes that a passing test does not show.** These fall into three
groups.

*Under concurrency and scale:*

- read-modify-write races that silently lose updates
- check-then-act logic that double-books, double-charges, or oversells
- migrations that break during the rolling deploy window
- retries without idempotency that duplicate side effects
- dual writes that leave two datastores permanently divergent
- a missing timeout that makes one slow dependency a total outage
- wall-clock timestamps used to order events across machines

*Against the kernel, on one machine:*

- a file updated in place, so a crash leaves half the old content and half the new
- a write reported as durable that never reached the disk, because nothing called `fsync`
- a rename that reaches the disk before the data, leaving the right name and no content
- a short `read` or `write` treated as the whole transfer
- a signal handler that calls `malloc` or `printf` and corrupts the heap
- a condition variable waited on with `if` instead of `while`
- a `fork` in a threaded process, where a library mutex stays locked forever

*When someone is attacking:*

- tokens and salts generated from `Math.random()`, which is unpredictable to a statistician
  and trivially predictable to an attacker
- encryption used where integrity was needed, so the ciphertext is malleable
- a signature that proves authorship and is silently assumed to prove freshness, leaving the
  endpoint replayable
- authentication mistaken for authorization — a logged-in user reading another tenant's row
- an internal service that trusts an `X-User-Id` header any pod can set
- a reset endpoint with no throttling, turning a six-digit code into a solved problem
- an attack that succeeds and leaves no audit record, because the only log is one the
  attacker can edit

This suite turns all three groups into **named, detectable catalogs** — 48 data hazards, 55
systems failures, and 64 security vulnerabilities. The planning, implementation, verification,
and review skills all check against them.

---

## Structure

```
skills/
├── engineer-workflow/SKILL.md      # Orchestrator: routes phases, owns shared state
├── planner/SKILL.md                # Recon → requirements → estimates → gates → phased plan.md
├── detail-planning/SKILL.md        # One phase → implementable spec in executor.md
├── implement/SKILL.md              # Spec → code, with robustness invariants
├── verify/SKILL.md                 # Code vs. spec, with evidence
├── code-review/SKILL.md            # 6 review modes, including /review-data, /review-security
├── system-design/                  # WHAT to build
│   ├── SKILL.md                    # 7-step method, design doc, red flags, proportionality
│   └── references/
│       ├── 01-design-method.md             # The steps in depth + question banks
│       ├── 02-estimation.md                # Powers of two, latency numbers, nines, formulas
│       ├── 03-scaling-ladder.md            # Single server → sharding, with triggers
│       ├── 04-building-blocks.md           # LB, cache, CDN, queues, rate limiting, IDs, protocols
│       ├── 05-database-selection.md        # Evaluation method, B-tree vs LSM, RUM conjecture
│       ├── 06-simplicity-and-design-laws.md # Six laws, three flaws, over-engineering tests
│       └── 07-reference-architectures.md   # Fan-out, chat, crawler, file sync, autocomplete…
├── data-systems-design/            # Whether it stays CORRECT
│   ├── SKILL.md                    # Decision tables, design record, language discipline
│   └── references/
│       ├── 01-reliability-scalability-maintainability.md
│       ├── 02-data-models.md
│       ├── 03-storage-and-retrieval.md
│       ├── 04-encoding-and-evolution.md
│       ├── 05-replication.md
│       ├── 06-partitioning.md
│       ├── 07-transactions.md
│       ├── 08-distributed-systems-faults.md
│       ├── 09-consistency-and-consensus.md
│       ├── 10-batch-processing.md
│       ├── 11-stream-processing.md
│       ├── 12-correctness-and-integrity.md
│       └── hazard-catalog.md       # 48 named hazards: signature → consequence → fix
├── systems-programming/            # Whether the CODE survives the kernel
│   ├── SKILL.md                    # 5 syscall rules, 7 file recipes, review scan, glossary
│   └── references/
│       ├── 01-file-descriptors-and-io.md    # fds, short counts, atomicity, fsync, mmap, locks
│       ├── 02-files-and-directories.md      # stat, links, rename, durable update, path safety
│       ├── 03-standard-io-and-buffering.md  # Buffer modes, flush vs fsync, fork duplication
│       ├── 04-processes-and-execution.md    # fork/exec/wait, exit status, races, daemon rules
│       ├── 05-signals.md                    # sigaction, async-signal-safety, EINTR, self-pipe
│       ├── 06-threads-and-concurrency.md    # Mutexes, condvars, lock order, fork with threads
│       ├── 07-ipc-and-sockets.md            # Pipes, framing, shared memory, fd passing
│       ├── 08-toolchain-and-machine-model.md # Assemble/link/load, symbols, storage classes
│       └── failure-catalog.md      # 55 named failures: signature → consequence → fix
└── security-engineering/           # Whether it survives an ATTACKER
    ├── SKILL.md                    # Threat model, security services, decision tables, security record
    └── references/
        ├── 01-threat-model-and-security-services.md
        ├── 02-cryptographic-primitives-and-modes.md
        ├── 03-randomness-and-key-management.md
        ├── 04-public-key-and-key-exchange.md
        ├── 05-integrity-hashes-and-macs.md
        ├── 06-signatures-and-authentication-protocols.md
        ├── 07-identity-certificates-and-pki.md
        ├── 08-transport-and-channel-security.md
        ├── 09-authentication-and-access-control.md
        ├── 10-intrusion-detection-and-audit.md
        ├── 11-malicious-software-and-availability.md
        ├── 12-perimeter-and-trusted-systems.md
        └── vulnerability-catalog.md # 64 named vulnerabilities: signature → consequence → fix
```

Each skill is a self-contained folder with a `SKILL.md`, which is the layout Claude Code,
Cursor, and compatible agents expect.

**The four gate skills split the problem deliberately.** `system-design` decides *what to
build* — scope, capacity, components, trade-offs. `data-systems-design` decides *whether it
stays correct* across machines — isolation, replication, ordering, hazards.
`systems-programming` decides *whether the code survives one kernel* — short reads, atomicity,
durability, signals, threads. `security-engineering` decides *whether it survives an
adversary* — trust boundaries, identity, secrets, untrusted input.

These are four different questions. Agents most often merge the fourth into the second by
mistake. Correctness analysis assumes that faults are random. Security analysis assumes that
an attacker chooses them. A queue consumer that is fully idempotent under duplicate delivery
can still be a replay vulnerability. Idempotency answers "did this happen twice" and not "did
the right party ask for it".

A new public service runs all four. Adding a retry to an existing call runs only the second.
A file-ingest pipeline or a daemon runs the third. Adding an authenticated endpoint runs only
the fourth.

---

## Installation

Copy the skill folders into your agent's skills directory:

```bash
# Claude Code / Cursor (user-level)
cp -r skills/* ~/.claude/skills/
cp -r skills/* ~/.cursor/skills/

# Project-level
cp -r skills/* .claude/skills/
```

On Windows (PowerShell):

```powershell
Copy-Item -Recurse -Force skills\* $HOME\.cursor\skills\
```

The agent finds each skill by its folder name. Install `data-systems-design`,
`systems-programming`, and `security-engineering` together with the rest. The hazard, failure,
and vulnerability scans in the other skills refer to those three catalogs by path.

---

## The pipeline

```
PLANNING → ARCHITECTURE → DESIGN → SYSTEMS → SECURITY → DETAIL PLANNING → IMPLEMENT → VERIFY → REVIEW
 plan.md    ##Architecture  ##Design  ##Systems  ##Security   executor.md      code      report   findings
```

| Command | What happens |
|---|---|
| `engineer-workflow: [idea]` | Start the full pipeline |
| `planner` | Recon the codebase, quantify requirements, estimate capacity, produce phased `plan.md` |
| `design` / `system design` | Run the 7-step design method, estimation, and simplicity pass |
| `estimate` | Just the back-of-the-envelope numbers |
| `data design` | Run the data-systems design gate and hazard scan |
| `systems` / `low level` | Run the systems gate: syscall rules, file recipes, failure scan |
| `security design` | Run the security gate: threat model and vulnerability scan of the design |
| `detail F1` | Expand phase F1 into a spec with contracts, failure modes, rollback |
| `implement F1` | Write the code, enforcing the robustness invariants |
| `verify F1` | Compare code to spec, with file-and-line evidence |
| `/review-diff` | Review the branch vs. main, plus PR/CI babysit status |
| `/review-uncommitted` | Catch work that looks finished but isn't |
| `/review` | Full repository audit |
| `/review-inscope` | Check the change against the stated task |
| `/review-data` | Deep data/concurrency/distribution hazard audit |
| `/review-security` | Deep security audit: threat model, then the vulnerability catalog |

Each phase stops when it's done and waits for you. One phase per cycle, by design.

### Example

```
> engineer-workflow: add a wallet with balance top-ups via Stripe

  → Planning: recon, requirements, explicit non-goals, phases F1–F4
  → Estimation: 40 top-ups/min peak, 2 KB/row, 1.2 GB/yr
      ⇒ single Postgres primary has years of headroom. No queue, no sharding.
  → Architecture gate: API contract, data model, write path walked end to end,
      trade-off table (ledger table vs. balance column → ledger, for auditability)
      Simplicity pass: Kafka removed (no number justifies it), Redis kept (p99 target)
  → Design gate triggers (money + persistence + external side effects):
      INV-1  balance never negative  → CHECK constraint + atomic UPDATE
      INV-2  one charge per request  → unique index on idempotency_key
      H-14 non-idempotent retry, H-06 side effect in transaction → mitigated in F2
  → Systems gate: not triggered. No file, process, or signal work in this change.
  → Security gate triggers (money + a Stripe webhook + a new endpoint):
      T-1  unauthenticated caller replaying a captured webhook
           → signature verified, 5-minute window, seen event_id cached (V-25, V-26)
      T-2  authenticated tenant reading another tenant's wallet
           → tenant predicate in the repository, negative test in F2 (V-44)
      Residual risk accepted: no per-request proof-of-possession on the session
      cookie. Mitigated by a 30-minute lifetime and server-side revocation (V-28)
  → STOP

> detail F2
  → Spec: files, signatures, failure-mode table, idempotency contract,
    isolation level, migration compatibility matrix, observability, rollback
  → STOP

> implement F2
  → Code + concurrency tests, validation run, deviations logged
  → STOP

> verify F2
  → ✗ FAIL: dedup uses a pre-flight SELECT outside the transaction (H-03/H-14)
    with the exact fix
```

---

## What the architecture gate produces

For a new system, a new datastore, a new component on the request path, or a scaling change:

- **Requirements** clarified before anyone proposes a solution, with explicit non-goals and
  labeled assumptions. Answering fast without clarifying is how you build the wrong system
  correctly
- **Capacity estimates** with derivations shown, each row marked measured or estimated, and
  the conclusion the numbers force ("sharding is not justified. The bottleneck at 10× is the
  `events` write path.")
- **An interface contract** written before the architecture, which is what makes the
  requirements concrete
- **A data model driven by access patterns**, with a datastore decision record listing the
  rejected alternative, the operability story, and the exit cost
- **Deep dives on the two or three components where the difficulty actually lives** — not
  even attention across every box
- **A bottleneck, failure-mode, and operations table**, including what saturates first, the
  degraded mode per dependency, and the next scale curve
- **A simplicity pass** that removes every component the numbers do not justify

---

## What the design gate produces

For anything touching persistence, concurrency, distribution, money, auth, or PII:

- **Requirements** as numbers — load parameters, p50/p99 (never averages, never averaged
  percentiles), RPO/RTO, and a consistency level chosen for a stated user-visible reason
- **A fault model** — which faults are tolerated, the resulting behavior, and blast radius
- **Decisions with rejected alternatives** — data model, storage engine, replication,
  partition key, isolation level, delivery semantics
- **Invariants with enforcement mechanisms** — "the application checks it first" is not a
  mechanism under concurrency
- **An evolution plan** — expand → migrate → contract, with a rollback path
- **"What breaks at 10x"** — the specific resource that saturates first
- **A hazard scan** against the catalog

It also enforces **language discipline**: phrases like "eventually consistent",
"we'll retry", "exactly once", "we'll keep them in sync", "add a cache", and
"use a distributed lock" are rejected unless accompanied by the actual mechanism.

---

## The hazard catalog

48 named hazards, each with a **detection signature** (what to grep for), a
**consequence**, and a **required fix** — grouped as:

| Group | Examples |
|---|---|
| A. Concurrency & transactions | H-01 lost update · H-02 write skew · H-03 uniqueness in app code · H-06 side effect in transaction |
| B. Schema & API evolution | H-09 breaking change in one deploy · H-11 unknown-field dropping · H-12 blocking DDL |
| C. Distributed calls & retries | H-14 non-idempotent retry · H-15 missing timeout · H-17 nested retries |
| D. Clocks, locks, leadership | H-20 wall-clock ordering · H-22 lock without fencing token · H-24 cross-channel race |
| E. Replication & partitioning | H-25 read-after-write from a replica · H-29 hot partition key · H-30 `hash mod N` |
| F. Streams, queues, derived data | H-32 dual write · H-33 unbounded queue · H-36 processing-time windowing |
| G. Reliability & operability | H-41 averages instead of percentiles · H-43 N+1 · H-44 cache without invalidation |

Every skill shares the catalog. `code-review` scans a diff with it, `verify` scans an
implementation, and `planner` scans a proposed design. The same defect therefore gets caught
at whichever stage it appears.

---

## What the systems gate produces

For a change that opens, writes, moves, or locks a file, that starts a process, that handles a
signal, or that shares memory between threads:

- **A durable-write recipe, named and applied** — temporary file in the same directory,
  `fsync`, `rename`, then `fsync` on the directory that holds it. "We write the file" is not a
  durability claim, and step 6 is the step engineers omit most often
- **Each pair of calls that forms one logical operation, replaced by the atomic call.** Use
  `O_APPEND` instead of seek-then-write, `O_EXCL` instead of test-then-create, and `pread`
  instead of seek-then-read
- **A loop around every `read` and `write`** that crosses a pipe, a socket, or a large buffer.
  A short count is normal behavior and not an error
- **A signal contract** — which signals the process handles, which functions each handler may
  call, and which work it defers to the main loop
- **A lock-order statement** for any code path that holds two mutexes
- **A language exposure check** — which of the five system call rules the runtime hides, and
  which it does not. No runtime makes two calls atomic. No runtime calls `fsync` for you.
- **A failure scan** against the catalog

The gate also names the pass structure for any program that transforms data. One question
decides it: does any output depend on input the program has not read yet? If yes, the program
needs two passes or a patch list.

---

## The failure catalog

55 named failures, each with a **detection signature**, a **consequence**, and a **required
fix** — grouped as:

| Group | Examples |
|---|---|
| A. File descriptors and I/O | S-01 unchecked `write` · S-03 short `read` as the whole file · S-06 `stat` before `open` · S-07 durable write with no `fsync` |
| B. Files, names, directories | S-11 write over a file in place · S-13 data reaching the disk after the rename · S-16 unchecked path from a user |
| C. Buffered streams | S-20 output a pipeline never shows · S-21 output duplicated after `fork` · S-22 stream and descriptor on one file |
| D. Processes | S-25 no `wait` for a child · S-28 command built from user input · S-29 descriptor leaked into a child |
| E. Signals | S-31 unsafe function in a handler · S-33 handler that loses `errno` · S-35 `SIGPIPE` with no handling |
| F. Threads | S-37 `pthread_cond_wait` inside an `if` · S-38 no lock order · S-42 `fork` in a process that has threads |
| G. IPC and sockets | S-45 stream with no framing · S-46 peer length used as an allocation size · S-47 network call with no timeout |
| H. Memory and toolchain | S-50 pointer to a returned local · S-51 size computed by multiplication · S-55 `struct` written to a socket |

Each entry states which languages it applies to. Most of them apply to Python, Go, Java, Rust,
and Node.js without change. A runtime hides `EINTR`. It does not hide the missing `fsync`, and
it does not make two calls atomic.

---

## What the security gate produces

For anything touching identity, authorization, secrets, cryptography, untrusted input, a new
reachable surface, or regulated data:

- **A named adversary with a stated capability** — a control chosen without an attacker in
  mind is a guess. The usual failure is a defense against the wrong attacker
- **The assets and trust boundaries**, so "internal" stops being a security argument
- **The security services owed** — authentication, access control, confidentiality, integrity,
  nonrepudiation, availability. Choose them per asset. Never assume them from the fact that
  something is encrypted
- **A mechanism per service**, at a named enforcement point in the code
- **A vulnerability scan** against the catalog, run against the design and again against the
  implementation
- **Detection** — which security events are emitted, with which fields, and where the audit
  trail lives
- **Residual risk, stated and owned** — the attacks this design does not stop, and why that is
  acceptable

It enforces the same language discipline as the design gate: "we validate the input", "it's
behind the VPN", "we sanitize it", "we encrypt it", and "internal tool, low risk" are rejected
unless accompanied by the boundary, the canonical form, and the mechanism.

---

## The vulnerability catalog

64 named vulnerabilities, each with a **detection signature**, a **consequence** traced to the
principle it violates, and a **required fix** — grouped as:

| Group | Examples |
|---|---|
| A. Cipher selection and modes | V-01 ECB · V-02 reused IV · V-05 keystream reuse · V-08 home-grown construction |
| B. Randomness, keys, and secrets | V-09 non-cryptographic PRNG · V-12 long-lived key used directly · V-14 secret in source |
| C. Integrity and message authentication | V-15 encryption as integrity · V-18 naive keyed hash · V-21 non-constant-time compare |
| D. Authentication protocols and freshness | V-25 no replay protection · V-28 reusable bearer credential · V-30 session not renewed |
| E. Identity, certificates, and trust | V-32 verification disabled · V-33 key from an unauthenticated channel · V-35 unauthenticated DH |
| F. Channel and transport security | V-37 unprotected sensitive traffic · V-38 downgrade permitted · V-42 handshake not bound |
| G. Access control and privilege | V-43 no authorization check · V-44 incomplete mediation · V-46 trust by network position |
| H. Passwords and credential storage | V-48 recoverable password · V-50 fast hash · V-53 unthrottled guessing · V-54 default credentials |
| I. Audit, detection, and logging | V-55 event not audited · V-56 modifiable audit trail · V-58 secrets in logs |
| J. Untrusted input, malicious code, availability | V-59 input reaching an interpreter · V-60 development backdoor · V-62 unbounded work |

The same sharing rule applies. The planner scans the design. `detail-planning` turns each
mitigation into a step with a negative test. `implement` enforces it. `verify` and
`code-review` then check that the mechanism exists where the record claimed it would.

---

## Design principles

**Proportionality.** Every skill has an explicit anti-over-engineering rule. A copy change
gets a three-bullet micro-plan and no hazard scan. A new source of truth gets the full
design record. Applying distributed-systems ceremony to a single-file fix is itself a
failure mode. Kanat-Alexander's version: the quality level of a design should be
proportional to how long the system will keep helping people.

**Numbers before architecture.** No component enters a design without an estimate or a
requirement behind it. "It should scale fine" is not a design, and neither is a diagram with
a queue nobody sized.

**Don't design for a future you can't measure.** The most common and disastrous design error
is predicting something about the future when you cannot know it. Design for measured load
with a stated growth rate. Make the next rung of the scaling ladder reachable. Don't build
it until a number says so.

**Assume the process dies between any two steps.** The kernel can stop a process between any
two calls. The power can fail between a write and the disk. Any operation that takes two calls
is not atomic, at any level of the stack. This is the same rule as check-then-act in a
database, one layer down.

**Maintenance cost outweighs implementation cost.** A design that's fast to build and
expensive to operate is a bad design. Every component added is a permanent tax on whoever
is on call.

**Evidence over assertion.** Verification and review findings must cite `file:line`.
"No critical issues found" is a valid result. Padding a review with invented nits trains
people to ignore reviews.

**Mechanisms over intentions.** Every invariant names the thing that enforces it. Every
retry names its idempotency key. Every cache names its invalidation trigger.

**Stop and ask.** Ambiguity that changes the design, irreversible operations, and
spec/code conflicts stop the pipeline rather than getting a guess.

---

## Using skills independently

Nothing requires the full pipeline:

- `code-review` reads the codebase and git state directly — no `plan.md` needed.
- `system-design` runs standalone for design work: "design a notification service", "will
  this scale?", "should we use Postgres or Cassandra here?", "estimate the storage for this".
  It's also useful as a review lens on an architecture someone else wrote.
- `data-systems-design` answers correctness questions on its own ("should this be
  serializable?", "is this partition key safe?").
- `systems-programming` answers low-level questions on its own: "how do I replace this file
  without losing it on a crash?", "why does my progress output vanish in a pipeline?", "is
  this signal handler safe?", "why is this `undefined reference` when the library is right
  there?". Its seven file-management recipes also work alone. Use them for a document-ingest,
  indexing, or RAG pipeline that must not lose or corrupt a file.
- `security-engineering` answers security questions on its own ("is this token design sound?",
  "how should we store these credentials?", "what can an attacker do with this endpoint?"), and
  works as a threat-modelling lens on a design someone else wrote.
- `planner` is useful alone for turning a vague request into a grounded plan.

---

## Migrating from the previous layout

Skills previously lived as flat files at the repository root. They are now folders under
`skills/`, matching the standard agent-skill layout:

| Before | After |
|---|---|
| `engineer-workflow.md` | `skills/engineer-workflow/SKILL.md` |
| `planner.md` | `skills/planner/SKILL.md` |
| `detail_planning.md` | `skills/detail-planning/SKILL.md` |
| `implement.md` | `skills/implement/SKILL.md` |
| `verify.md` | `skills/verify/SKILL.md` |
| `code-review.md` | `skills/code-review/SKILL.md` |
| — | `skills/data-systems-design/` (new) |
| — | `skills/system-design/` (new) |
| — | `skills/systems-programming/` (new) |
| — | `skills/security-engineering/` (new) |

If you installed the old flat files, remove them before you install the new folders.
Otherwise the agent loads two versions of the same skill.

---

## Dependencies

None. These are Markdown skill definitions for AI agents.

## Contributing

Fork, branch, PR. New hazards, failures, and vulnerabilities are welcome. Follow the format
of the catalog you add to (signature → consequence → fix) and cite the source of the failure
mode. In the failure catalog, state which languages each entry applies to. In the
vulnerability catalog, mark guidance that post-dates the 4th edition with the **Modern** tag
rather than attributing it to Stallings.

## License

MIT

## Attribution

Concepts, terminology, methodology, and reference numbers come from:

- [*Designing Data-Intensive Applications*][ddia] — Martin Kleppmann (O'Reilly, 2017).
  Page references throughout `data-systems-design` refer to that edition.
- [*Cryptography and Network Security: Principles and Practices*][stallings] — William
  Stallings (Prentice Hall, 4th ed., 2005). Security services and mechanisms, the attack
  taxonomy, and the reference-monitor and audit-record models. Page and section references
  throughout `security-engineering` refer to that edition. Guidance that post-dates it is
  marked **Modern** rather than attributed to the book.
- [*System Design Interview: An Insider's Guide*][xu] — Alex Xu (2020). The design
  framework, back-of-the-envelope estimation, and the scaling progression.
- [*Grokking the System Design Interview*][grok] — Design Gurus. The seven-step process and
  the building-block catalog.
- [*Database Internals*][dbi] — Alex Petrov (O'Reilly, 2019). Database evaluation
  methodology, storage engine trade-offs, and the RUM conjecture.
- [*Code Simplicity: The Science of Software Development*][cs] — Max Kanat-Alexander
  (O'Reilly, 2012). The laws of software design and the anti-over-engineering discipline.
- [*Advanced Programming in the UNIX Environment*][apue] — W. Richard Stevens and Stephen A.
  Rago (Addison-Wesley, 3rd ed., 2013). The system call semantics, atomicity rules, file
  management, signals, and threads throughout `systems-programming`.
- [*Systems Programming*][donovan] — John J. Donovan (McGraw-Hill, 1972). The machine model,
  the design procedure for a system program, and the four functions of a loader.

This repository contains original prose that applies those concepts to agent workflows. It is
not a reproduction of any of the books. Read them.

[ddia]: https://dataintensive.net/
[stallings]: https://williamstallings.com/Cryptography/
[xu]: https://www.systemdesigninsider.com/
[grok]: https://www.designgurus.io/course/grokking-the-system-design-interview
[dbi]: https://www.databass.dev/
[cs]: https://www.codesimplicity.com/
[apue]: https://www.apuebook.com/
[donovan]: https://archive.org/details/systemsprogrammi0000dono
